### GUIA DE ESTUDIO PARCIAL 2 


# POSIBLE PARCIAL 
Creación de API para estudiantes del siguientes semestres
* El usuario se registra y obtiene un API KEY 
* El usuario envia el TOKEN como header para acceder a ciertos recursos
* La API tiene endpoints públicos y privados
* La API retorna recursos multimedia 

Crear proyecto 

```
nest new <project_name>
```

Vamos a utilizar docker con postgres, si tu computador no soporta el proceso de docker 
puedes usar algún servicio cloud de postgres por ejemplo neontech, bd postgres dentro de vercel

Agregar variables de entorno base de datos para docker
* crea archivo .env
* docker puede acceder al archivo .env asi no este configurado en nest las variables de entorno

```
DB_PASSWORD=clave
DB_NAME=nombre
```
## Levantar base de datos local docker
ejecuta docker compose para desplegar la base de datos
* va buscar el archivo docker-compose.yml 
``` 
docker-compose up -d
```

valida en tu gestor de base de datos preferido si te puedes conectar
* db url y conectarse en pgadmin, tableplus, dbbeaver, etc.. 
```
localhost:5432
user: postgres
password: <la misma del .env>
```

remover carpeta de /postgres en el .gitignore para no subir información
innecesaria al repositorio. 

## Conectar Postgres con Nest

### variables de entorno
Para empezar vamos a preparar Nest para que pueda recibir variables de entorno
* instalar modulo de config 
```npm i @nestjs/config```

* configurar el archivo app.module.ts
```
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()], // módulo de variables de entorno
})
export class AppModule {}
```

### agregar drivers bases de datos
* el último driver va variar dependiendo la base de datos que uses
```
npm i @nestjs/typeorm typeorm pg
```

agregar variables de entorno relacionadas a DB faltantes
```
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
```

* configurar app.module.ts, solamente habrá un forRoot en el proyecto
```
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    ConfigModule.forRoot(),
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST,
      port: +process.env.DB_PORT!,
      database: process.env.DB_NAME,
      username: process.env.DB_USERNAME,
      password: process.env.DB_PASSWORD,
      autoLoadEntities: true,
      synchronize: true, // SOLO EN DESARROLLO
    })
  ],
})
export class AppModule {}
```

###  Genera tus recursos 

Dependiendo tu problema/proyecto ya puedes generar los recursos
``` nest g res <recurso>```

TypeORM va buscar la entity luego configurarlo para que sea tomado 
como tabla de base de Datos
* ir a recurso.entity.ts

```
import { Column, Entity, PrimaryGeneratedColumn } from "typeorm";

@Entity()
export class Product {

    @PrimaryGeneratedColumn('uuid')
    id: string;

    @Column('text', {
        unique: true
    })
    title: string;

    @Column({
        type: 'text',
        nullable: true
    })
    description: string;

    @Column('text', {
        array: true
    })
    sizes: string[]

    @Column('bool', {
        default: false
    })
    active: boolean;

}

```

### Configura Nest para que lea la configuración del recurso

* importar la entidad junto con typeOrm ForFeature en recurso.module.ts

```
import { Module } from '@nestjs/common';
import { ProductsService } from './products.service';
import { ProductsController } from './products.controller';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Product } from './entities/product.entity';

@Module({
  controllers: [ProductsController],
  providers: [ProductsService],
  imports: [
    TypeOrmModule.forFeature([Product])
  ]
})
export class ProductsModule {}

```

Al agregar este módulo se actualiza la bd.

### Configurar DTO

instalar dependencias para hacer validaciones
```
npm i class-validator class-transformer
```

* Configurar main.ts

```
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.setGlobalPrefix('api')
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // remueve lo que no esta en el DTO
      forbidNonWhitelisted: true // retorna bad request 
    })
  )
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();

```

configurar DTO
*  buscar /dto/create-recurso.dto.ts

```
import { IsArray, IsBoolean, IsNumber, IsOptional, IsPositive, 
    IsString, MinLength } from "class-validator";

export class CreateProductDto {

    @IsString()
    @MinLength(1)
    title: string;

    @IsPositive()
    @IsNumber()
    @IsOptional()
    price?: number;
    
    @IsOptional()
    @IsString()
    description?: string;
    
    @IsString({each: true})
    @IsArray()
    sizes: string[];
    
    @IsOptional()
    @IsBoolean()
    active?: boolean;


}
```

### Insertar usando typeORM

* en el servicio usar el patrón repositorio en el constructor con tu recurso

```
constructor(
    @InjectRepository(Product)
    private readonly productRepository: Repository<Product>
  ){}
```

* insertar en nuestra función

```
async create(createProductDto: CreateProductDto) {
    try {
      const product = this.productRepository.create(createProductDto)
      await this.productRepository.save(product)
      return product;
    } catch (error) {
      this.handleDBExceptions(error);
    }
  }
```

* código para manejo errores

```
private handleDBExceptions(error: any){
    if(error.code === '23505'){
        throw new BadRequestException(error.detail)
      }
      this.logger.error(error)
      throw new InternalServerErrorException('Unexpected error, check logs ')
  }
```


- Decoradores de utilidad antes de insertar o antes de actualizar
* En nuestra entity tenemos disponibles decoradores para configurar transformaciones en los datos previos a guardar.
* En este ejemplo vamos a guardar todos en mayusculas. en recurso.entity.ts

```
@BeforeInsert()
    checkTitleInsert(){
        this.title = this.title.toUpperCase()
    }
```

### Consultar y borrar 

Consultar todos recurso.service
```
async findAll() {
   try {
    return await this.productRepository.find({})
   } catch (error) {
    this.handleDBExceptions(error)
   }
  }
```

consultar por ID
Controlador
* Es posible usar el ParseUUIDPipe para validar entradas de parametros
```
 @Get(':id')
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.productsService.findOne(id);
  }
```
* servicio

```
async findOne(id: string) {
    try {
      const product = await this.productRepository.findOneBy({id})
      if(!product){
        throw new NotFoundException(`Producto con el id ${id} no fue encontrado`)
      }
      return product;
    } catch (error) {
      this.handleDBExceptions(error)
    }
  }
```

* borrar  
controlador
```
@Delete(':id')
  remove(@Param('id', ParseUUIDPipe) id: string) {
    return this.productsService.remove(id);
  }
```


* servicio
```
 async remove(id: string) {
    try {
      const product = await this.findOne(id)
      if(product){
        await this.productRepository.remove(product)
      }
    } catch (error) {
       this.handleDBExceptions(error)
    }
  }
  ```

# RELACIONES !! 

En este ejemplo vamos agregar una nueva table para agregar la imagen

* crear en la carpeta entity un nuevo archivo, en este caso no es necesario un CRUD nuevo. Analice su caso para ver si lo requiere


```
import { Column, Entity, PrimaryGeneratedColumn } from "typeorm";

@Entity()
export class ProductImage {

    @PrimaryGeneratedColumn()
    id: number;


    @Column('text')
    url: string;
}

```

En este momento no se ha sincronizado, es importante volver al modulo y agregar los nuevos elementos al arreglo de typeorm

```
import { TypeOrmModule } from '@nestjs/typeorm';
import { Product } from './entities/product.entity';
import { ProductImage } from './entities/product-image.entity';

@Module({
  controllers: [ProductsController],
  providers: [ProductsService],
  imports: [
    TypeOrmModule.forFeature([Product, ProductImage])
  ]
})
export class ProductsModule {}
```

### one to many & many to one
Ahora es necesario indicarle a cada entidad que tiene una relación con la otra entidad.
Para facilitar el aprendizaje es mejor tener las dos entidades en pantalla dividida

Product Entity
```
    @OneToMany(
        ()=> ProductImage,
        (productImage) => productImage.product,
        {cascade: true} //evitar true para q no haya huerfanos
    )
    images?: ProductImage
```
ProductImage Entity
```
@ManyToOne(
        () => Product,
        (product)=> product.images
    )
    product: Product
```

### Crear recursos con relaciones
* primero ajustar el dto en el crear 
```
 @IsString({each: true})
    @IsArray()
    @IsOptional()
    images?: string[]
```

Este ajuste les va romper el create, se arregla con un arreglo sin images
```
  const product = this.productRepository.create({...createProductDto, images: []})
```

* recuerden que para cualquier interacción con la base de datos necesitamos usar el patrón repositorio e inyección de dependencias 

```
 constructor(
    @InjectRepository(Product)
    private readonly productRepository: Repository<Product>,
    @InjectRepository(ProductImage)
    private readonly productImageRepository: Repository<ProductImage>,
    ){}
```

* Luego de agregar este repositorio nos va quedar de la siguiente forma, nos esta creando la relación 

```
  async create(createProductDto: CreateProductDto) {
    try {
      const {images = [], ...productDetails} = createProductDto;
      const product = this.productRepository.create({
        ...productDetails, 
        images: images.map(img => this.productImageRepository.create({url: img})) })
      await this.productRepository.save(product)
      // return product para este caso no quiero el id de la img, quiero solamente el string y lo ajusto 
      // por este motivo lo retorno de esta forma
      return {...product, images};
    } catch (error) {
      this.handleDBExceptions(error);
    }
  }
```

* Configurar que carguen las relaciones en las consultas del ORM
Enfocarnos en la propiedad relations, donde podemos agregar a la respuesta el número de relaciones que queramos

```
 async findAll(pagination: PaginationDto) {
    const {limit=10, offset=0} = pagination;
   try {
    return await this.productRepository.find({
      take: limit,
      skip: offset,
      relations: {
        images: true
      }
    })
   } catch (error) {
    this.handleDBExceptions(error)
   }
  }
  ```
