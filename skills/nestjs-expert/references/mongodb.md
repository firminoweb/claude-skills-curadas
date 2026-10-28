# MongoDB with Mongoose

Baseline: `@nestjs/mongoose` 12, `mongoose` 9 (Node ≥ 20.19), Oct 2026. Prisma 7 doesn't support MongoDB; for MongoDB use Mongoose. A project already on Prisma 6 + MongoDB can stay there, but don't start new ones on it.

The rules in `SKILL.md` still apply (never return secrets, authorization in the query, validation at the edge). Only the data layer changes.

## Setup

```bash
npm i @nestjs/mongoose@12 mongoose@9
```

```ts
// app.module.ts
MongooseModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    uri: config.getOrThrow<string>('MONGODB_URI'),
    autoIndex: config.get('NODE_ENV') !== 'production', // build indexes at deploy, not at boot (below)
    serverSelectionTimeoutMS: 5_000,
  }),
}),
```

```ts
// main.ts, before the app is created
mongoose.set('sanitizeFilter', true); // strips $-operators smuggled into filters ({ email: { $ne: null } })
```

## Schema

```ts
// users/user.schema.ts
@Schema({
  timestamps: true,
  toJSON: {
    virtuals: true,
    versionKey: false,
    transform: (_doc, ret: Record<string, unknown>) => { delete ret._id; delete ret.passwordHash; return ret; },
  },
})
export class User {
  @Prop({ required: true, unique: true, lowercase: true, trim: true }) email: string;
  @Prop({ required: true }) name: string;
  @Prop({ required: true, select: false }) passwordHash: string; // excluded unless .select('+passwordHash')
  @Prop({ type: String, enum: ['user', 'admin'], default: 'user' }) role: string;
}
export type UserDocument = HydratedDocument<User>;
export const UserSchema = SchemaFactory.createForClass(User);
UserSchema.index({ role: 1, createdAt: -1 });

// users/users.module.ts
imports: [MongooseModule.forFeature([{ name: User.name, schema: UserSchema }])],
```

`select: false` + the `toJSON` transform are two layers against leaking the hash. `.lean()` results skip `toJSON`, so project fields explicitly there.

## Service

```ts
@Injectable()
export class UsersService {
  constructor(@InjectModel(User.name) private readonly users: Model<User>) {}

  async create(dto: CreateUserDto) {
    try {
      const user = await this.users.create({ email: dto.email, name: dto.name, passwordHash: await hash(dto.password) });
      return user.toJSON();
    } catch (e) {
      if (e instanceof mongo.MongoServerError && e.code === 11000) throw new ConflictException('Email already registered');
      throw e;
    }
  }

  async findOne(id: string) {
    const user = await this.users.findById(id).select('email name role createdAt').lean().exec();
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }
}

// controller: validate ObjectId params
@Get(':id') findOne(@Param('id', ParseObjectIdPipe) id: Types.ObjectId) {}
```

- `.lean()` for reads that only return data: plain objects, much faster; no virtuals, getters or `toJSON`.
- `.exec()` gives a real promise and better stack traces.
- Ownership in the filter: `findOne({ _id: id, ownerId: user.userId })`, never fetch then compare.
- Updates: `findOneAndUpdate(filter, { $set: dto }, { returnDocument: 'after', runValidators: true })`. Always `$set` with whitelisted DTO fields; never pass the raw body as the update.

## Errors

One filter, same idea as the Prisma filter:

| Error | Meaning | HTTP |
|-------|---------|------|
| `MongoServerError` code `11000` | Duplicate key (unique index) | 409 |
| `mongoose.Error.CastError` | Invalid id/type in a filter | 400 |
| `mongoose.Error.ValidationError` | Schema validation failed (should have been caught by the DTO) | 400 |
| `mongoose.Error.DocumentNotFoundError` | `orFail()` found nothing | 404 |

Anything else stays a generic 500.

## Modeling

- **Embed** what is read together and bounded (address, order items). **Reference** what grows without limit or is shared (comments, users). A document is capped at 16 MB, and an array that keeps growing slows every write.
- Shape documents around the main queries; duplicate a few fields (author name on a post) when it removes a lookup, and update them when the source changes.
- `populate()` runs one extra query per path (not N+1), but it's still a round trip; `$lookup` in an aggregation does it in the database.
- Store money as integer cents or `Decimal128`, never floating-point.

## Indexes and migrations

- Every query filter/sort in a hot path has an index (`schema.index`). Check with `.explain('executionStats')`: `COLLSCAN` on a big collection is the problem.
- `autoIndex: false` in production; build indexes as a deploy step (`await Model.syncIndexes()` in a script, or `createIndexes`), since building on boot slows startup and runs on every instance.
- MongoDB has no schema migrations, but data still needs them (rename field, backfill): `migrate-mongo` scripts, run as the pre-deploy command, written to work while old and new code run side by side.

## Transactions

Need a replica set (Atlas always is; locally run a single-node replica set).

```ts
import type { Connection } from 'mongoose'; // `import type`: see the note below

constructor(@InjectConnection() private readonly connection: Connection) {}

await this.connection.transaction(async (session) => {
  await this.orders.create([{ ...order }], { session });
  await this.stock.updateOne({ _id: sku, qty: { $gte: n } }, { $inc: { qty: -n } }, { session });
});
```

In an ESM project (`"type": "module"`), a value import `import { Connection } from 'mongoose'` crashes at boot ("Named export 'Connection' not found"): mongoose is CommonJS and Node doesn't detect that export, and `emitDecoratorMetadata` turns the constructor type into a runtime reference. `import type` avoids it and works in CommonJS too. `Model`, `Types` and `mongo` are fine as named imports.

`connection.transaction()` retries transient transaction errors. Pass `session` to **every** operation inside, or that operation runs outside the transaction. Prefer single-document atomic updates (`$inc`, conditional filter) when they suffice: they need no transaction.

## Pagination

Every list has `limit`. For big collections use range pagination: `find({ _id: { $lt: lastId } }).sort({ _id: -1 }).limit(20)` instead of deep `skip`.

## Tests and health

- Unit: mock the model with `getModelToken(User.name)`.
- E2E: `mongodb-memory-server` (`MongoMemoryReplSet` when the code uses transactions) or `@testcontainers/mongodb`; clean collections between tests.
- Health: `MongooseHealthIndicator` from `@nestjs/terminus` (`pingCheck('mongodb')`).
- Hosting: MongoDB Atlas (managed, replica set by default). Railway has a MongoDB template; Render has no managed MongoDB.

## Outdated habits

| Outdated | Current (Mongoose 9) |
|----------|----------------------|
| `{ new: true }` on `findOneAndUpdate` | `{ returnDocument: 'after' }` |
| `pre('save', function (next) { ...; next() })` | `async function` without `next` |
| `FilterQuery<T>` | `QueryFilter<T>` |
| Callbacks (`Model.find({}, cb)`) | `await ...exec()` |
| `useNewUrlParser`, `useUnifiedTopology` | Removed; just the URI |
| Update pipeline `updateOne(f, [{ $set }])` working silently | Throws unless `{ updatePipeline: true }` |
| Prisma for new MongoDB projects | Mongoose (Prisma 7 has no MongoDB) |
