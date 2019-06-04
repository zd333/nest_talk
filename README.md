---
marp: true
---

# REST API on steroids for Angular developer (and not only)

TODO: image with big Nest and Angular logo (plus React, VUE, JS)

---

![bg contain](./p-fortran.png)

---

# When frontender picks tech stack for API

Do I go serverless?

Do I use one of those ugly alien (non-JS) languages/frameworks?

I want to have JS everywhere!

---

# Choose among over 9000 Node.js frameworks

[Express](https://expressjs.com/)

[koa](https://koajs.com/)

[hapi](https://hapijs.com/)

[LoopBack](https://loopback.io/)

[restify](http://restify.com/)

[Sails](https://sailsjs.com/)

[Feathers](https://feathersjs.com/)

[Adonis](https://adonisjs.com/)

[Total](https://www.totaljs.com/)

...

---

# Why I choose nest (and you should consider too)

Opinionated modular design/architecture (maintainability and scalability battle tested)

Typescript

Familiar to Angular developer

[Nx](https://nx.dev/)

Excellent documentation

Great community and support

Express under the hood (out of the box)

Compatibility with other libraries (e.g. Fasify or even custom adapters)

---

# More about awesome Nx

[https://nx.dev/](https://nx.dev/)

NestJS

React

Jest, Cypress

Angular NGRX + missing things (data persistance)

Angular Console

---

# Building blocks

Modules

Controllers

Providers (Services)

Pipes

Guards

Interceptors

Exception filters

Middlewares

---

# Controllers

```typescript
@Controller('tenants')
export class TenantsController {

  @Post()
  public async create(@Body() dto: CreateTenantDto): Promise<number> {
    const dbDocument = await this.tenantsDbConnector.create(dto);

    return dbDocument.id;
  }
}
```

[https://example.com/tenants](https://example.com/tenants)

---

# Route parameters

```typescript
@Controller('employees')
export class EmployeesController {

  @Get(':id')
  public async getById(@Param() { id }): Promise<EmployeeDetailsDto> {
    const dbDocument = await this.employeesDbConnector.getById(id);

    if (!dbDocument) {
      throw new NotFoundException();
    }

    return convertDocumentToDto({
      dbDocument,
      dtoConstructor: EmployeeDetailsDto
    });
  }
}
```

[https://example.com/employees/100500](https://example.com/employees/100500)

---

# Global prefix, custom routes

```typescript
// main.ts

app.setGlobalPrefix('api/v1');

// ...

@Controller('employees')
export class EmployeesController {

  @Post('register')
  public async register(
    @Body() dto: RegisterEmployeeDto
  ): Promise<RegisteredEmployeeDto> {
    // ...
  }
}
```

[https://example.com/api/v1/employees/register](https://example.com/api/v1/employees/register)

---

# DB

[TypeORM](https://github.com/typeorm/typeorm)

[Mongoose](https://github.com/Automattic/mongoose)

---

# Mongoose - configuration

```typescript
const mongoConnStr = process.env.MONGO_CONNECTION_STRING;

@Module({
  imports: [
    MongooseModule.forRoot(mongoConnStr),
    AuthenticationModule,
    EmployeesModule,
    TenantsModule
  ]
})
export class AppModule {}
```

---

# Mongoose - schema definition

```typescript
const schemaDefinition: SchemaDefinition = {
  name: { type: String, required: true },
  roles: [{ type: String, enum: allAppAccessRoles, required: false }]
  isActive: { type: Boolean, required: true },
  login: { type: String, required: true },
  passwordHash: String,
};

const schema = new Schema(schemaDefinition);
schema.pre('save', passwordHashingHook);

export const EmployeeSchema = schema;
```

---

# Mongoose - usage

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([
      { collection: 'Employees', schema: EmployeeSchema }
    ])
  ]
})
export class EmployeesModule {}
// ...
let employeeModel: Model<Employee>;
// ...
const model = new employeeModel({
  name: 'John Snow',
  roles: ['queenSlayer'],
  isActive: false,
  login: 'sweety',
  password: '123456'
});

await model.save();
// ...
const dbDocument = await employeeModel.findById('100500').exec();
```

---

# Providers (Services), DI

```typescript
@Injectable()
export class TenantsDbConnectorService {

  constructor(
    private readonly employeesDbConnectorService: EmployeesDbConnectorService,
    @InjectModel('Tenants') private readonly tenantModel: Model<Tenant>
  ) {}

  public async create(dto: CreateTenantDto): Promise<TenantDocument> {
    const doc = new this.tenantModel(dto);

    await doc.save();

    return doc;
  }

  public async getById(id: string): Promise<TenantDocument | null> {
    return await this.tenantModel.findById(id).exec();
  }
}
```

---

# Declarative validation

[https://github.com/typestack/class-validator](https://github.com/typestack/class-validator)

```typescript
export class RegisterEmployeeDto {
  @MinLength(3)
  public readonly name: string;

  @IsOptional()
  @ArrayUnique()
  @IsIn(allAppAccessRoles, { each: true })
  @NotEquals('_ADMIN', { each: true })
  public readonly roles?: Array<AppAccessRoles>;

  @Validate(IsNotExpiredJwtTokenValidator)
  public readonly registrationToken: string;
}
```

---

# Custom validators

```typescript
@ValidatorConstraint()
@Injectable()
export class IsNotExpiredJwtTokenValidator implements ValidatorConstraintInterface {

  constructor(private readonly jwt: JwtService) {}

  public validate(value: string): boolean {
    return this.jwt.verify(value);
  }

  public defaultMessage(): string {
    return '$value must be valid and not expired JWT token';
  }
}
```

---

# Validation usage

```typescript
@Post('register')
@UsePipes(new ValidationPipe())
public async register(
  @Body() dto: RegisterEmployeeDto,
): Promise<void> {
  // ...
}
// ...

app.useGlobalPipes(
  new ValidationPipe({
    forbidUnknownValues: true,
  }),
);
```

---

# Authentication and guards

```typescript
@Module({
  imports: [
    PassportModule.register({ defaultStrategy: 'jwt' }),
  ],
})
export class AuthenticationModule {}
// ...

@UseGuards(AuthGuard())
@Put(':id')
public async update(
  @Param() { id }: string,
  @Body() dto: UpdateEmployeeDto,
): Promise<void> {
  // ...
}
```

---

# Roles-based access control

[https://github.com/nestjs-community/nest-access-control](https://github.com/nestjs-community/nest-access-control)

```typescript
export const appRoles = new RolesBuilder();

appRoles.grant('_BASIC')
  .readAny('tenant')
  .createOwn('employee');

appRoles.grant('_ADMIN')
  .extend('_BASIC')
  .updateAny('employee')
  .deleteAny('employee');
// ...

@Module({
  imports: [
    AccessControlModule.forRoles(appRoles),
  ],
})
export class AppModule {}
```

---

# Authorization roles usage

```typescript
@UseGuards(
  AuthGuard(),
  ACGuard,
)
@UseRoles({
  resource: 'employee',
  action: 'update',
  possession: 'any',
})
@Put(':id')
public async update(
  @Param() { id }: string,
  @Body() dto: UpdateEmployeeDto,
): Promise<void> {
  // ...
}
```

---

# Middlewares

```typescript
@Injectable()
export class AddClinicContextMiddleware implements NestMiddleware {
  constructor(private readonly clinicsDbConnector: ClinicsDbConnectorService) {}

  public resolve(): MiddlewareFunction {
    return async (req: AppRequest, res, next) => {
      const hostName = req.header('host');
      const targetClinicId = await this.clinicsDbConnector.getClinicIdByHostName(hostName);

      req.body.targetClinicId = targetClinicId;
      next();
    };
  }
}
// ...

export class AppModule implements NestModule {
  public configure(consumer: MiddlewareConsumer): void {
    consumer.apply(AddClinicContextMiddleware).forRoutes({
      path: '*',
      method: RequestMethod.ALL,
    });
  }
}
```

---

# Above and beyond

Websockets

GraphQL

Microservices

Swagger

Sentry

...

---

# Thank You so much

Presentation [https://github.com/zd333/nest_talk](https://github.com/zd333/nest_talk)
![QR code with link to presentation](./presentation.png)

Additional useful info
TODO:
