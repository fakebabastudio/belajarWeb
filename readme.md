## Authentication - Lupa Password Feature

### 1. Membuat Fitur Lupa Password

Pada Materi ini kita akan membahas bagaimana membuat fitur lupa password. Pada fitur ini biasanya backend akan mengirim link ke email untuk lupa password. 

*Langkah 1*

Petama kita akan instalasi package nodemailer untuk mengirimkan email pada nodejs.
```tsx
npm install --save @nestjs-modules/mailer nodemailer
npm install --save-dev @types/nodemailer
npm install --save handlebars   // library JavaScript yang digunakan untuk memfasilitasi proses templating di sisi klien. Dengan menggunakan Handlebars.js, Anda dapat menggabungkan data dengan template HTML untuk menghasilkan output HTML yang lebih dinamis dan fleksibel
```

 

Kemudian kita buat feature mail

```tsx
npx nest module app/mail
npx nest service app/mail
```

*Langkah 2* 
Buatlah akun pada https://mailtrap.io/ untuk membuat smtp server dummy. 
Pada proses development ini kita akan menggukan mailtrap untuk menerima email pada saat lupa password.
Setelah membuat akun silakan buka url https://mailtrap.io/

![Alt text](https://drive.google.com/uc?id=11Q4KKdE52G0juJQJtM-2kOw2AuIBpo6O)


Kemudian buka My Inbox dan pada bagian integrations pilih nodemailer

![Alt text](https://drive.google.com/uc?id=1MYJcgHoMaRhyqJ0nhMnfFR1MhgYmhwZK)

```tsx
var transport = nodemailer.createTransport({
  host: "sandbox.smtp.mailtrap.io",
  port: 2525,
  auth: {
    user: "116b44e4fce785",
    pass: "0a66404****"
  }
})
```

Pada kode di atas kita akan mendapatkan konfigurasi untuk nanti simpan di nestjs. Terlihat pada bagian pass ada *** untuk mendapatkan string lengkap nya silahkan kalian klik button copy di sebelah kanan.

*Langkah 3*
Konfigurasi mail module

*mail.module.ts*

```tsx
import { MailerModule } from '@nestjs-modules/mailer';
import { HandlebarsAdapter } from '@nestjs-modules/mailer/dist/adapters/handlebars.adapter';
import { Module } from '@nestjs/common';
import { MailService } from './mail.service';
import { join } from 'path';

@Module({
  imports: [
    MailerModule.forRoot({
      transport: {
        host: 'sandbox.smtp.mailtrap.io', //sesuaikan konfigurasi 
        port: 2525,
        auth: {
          user: '116b44e4fce785',  //sesuaikan user
          pass: '0a66404e26**', //sesuaikan password 
        },
      },
      defaults: {
        from: '"No Reply" <noreply@example.com>',
      },
      template: {
        dir: join(__dirname, 'templates'),  // template akan di ambil dari handlebar yang ada pada folder templates
        adapter: new HandlebarsAdapter(),
        options: {
          strict: true,
        },
      },
    }),
  ],
  providers: [MailService],
  exports: [MailService], // 👈 export  mailService agar bisa digunakan di luar module mail
})
export class MailModule {}
```


*Langkah 4*

Membuat folder template pada mail 

![Alt text](https://drive.google.com/uc?id=1nqf3k_PQZ1Z7zE55IkWypr3JB8WfJHpP)

*lupa_password.hbs*

```tsx
<p>Hey {{ name }},</p>
<p>Please click below to confirm your email</p>
<p>
    <a  href="{{ link }}">Klik</a>
</p>
```


Secara default NestJs hanya mendistibuskan mengcompile file .js dan .ts pada saat build. kita lihat bahwa template menggukaan extensi .hbs. Maka kita harus tambahkan konfiguasi pada nest-cli.json  agar nestjs dapat mendistibusikan .hbs.



*nest-cli.json*

```tsx
"compilerOptions": {
    "assets": ["app/mail/templates/**/*"],
    "watchAssets": true
  },
```


Tambahkan file di atas pada mest-cli.json, sehingga menjadi

```tsx
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "compilerOptions": {
    "assets": ["app/mail/templates/**/*"],
    "watchAssets": true
  },
  "sourceRoot": "src"
}
```


*Langkah 5*

Membuat DTO MailResetPassword dan membuat method pada mail service

*mail.dto.ts*

```tsx
export class MailResetPasswordDto {
  link: string;
  name: string;
  email: string;
}
```

*mail.service.ts*

```tsx
import { Injectable } from '@nestjs/common';
import { MailerService } from '@nestjs-modules/mailer'; //import MailerService
import { MailResetPasswordDto } from './mail.dto';

@Injectable()
export class MailService {
  constructor(private mailService: MailerService) {}

  async sendForgotPassword(payload: MailResetPasswordDto) {
    await this.mailService.sendMail({
      to: payload.email,
      subject: 'Lupa Password', // subject pada email
      template: './lupa_password',  // template yang digunakan adalah lupa_password, kita bisa memembuat template yang lain
      context: {
        link: payload.link,
        name: payload.name,
      },
    });
  }
}
```


*Langkah 6*

import mailModule pada auth module

*auth.module.ts*

```tsx
import { Module } from '@nestjs/common';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './auth.entity';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwt_config } from 'src/config/jwt.config';
import { JwtStrategy } from './jwt.strategy';
import { MailModule } from '../mail/mail.module';
import { ResetPassword } from './reset_password.entity';

@Module({
  imports: [
    TypeOrmModule.forFeature([User, ResetPassword]),
    PassportModule.register({
      defaultStrategy: 'jwt',
      property: 'user',
      session: false,
    }),
    JwtModule.register({
      secret: jwt_config.secret,
      signOptions: {
        expiresIn: jwt_config.expired,
      },
    }),
    MailModule, // import disini
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
})
export class AuthModule {}
```


*Langkah 7* 

Kemudian kita akan membuat tabel reset_password untuk menyimpan user_id dan token dari lupa passwod.

Buatlah file reset_password.entity.ts pada folder Auth

*reset_password.entity.ts*
```tsx
import {
  Entity,
  BaseEntity,
  PrimaryGeneratedColumn,
  Column,
  ManyToOne,
  JoinColumn,
} from 'typeorm';
import { User } from './auth.entity';

@Entity()
export class ResetPassword extends BaseEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @ManyToOne(() => User)  // relasikan many to one dengan table user
  @JoinColumn({ name: 'user_id' })
  user: User;

  @Column({ nullable: true })
  token: string;

  @Column({ type: 'datetime', default: () => 'CURRENT_TIMESTAMP' })
  created_at: Date;

  @Column({ type: 'datetime', default: () => 'CURRENT_TIMESTAMP' })
  updated_at: Date;
}
```


Karena ada relasi dengan tabel user, maka kita akan update pada file auth.entity.ts

```tsx
import {
  Entity,
  BaseEntity,
  PrimaryGeneratedColumn,
  Column,
  ManyToOne,
  JoinColumn,
  OneToMany,
} from 'typeorm';
import { ResetPassword } from './reset_password.entity';

@Entity()
export class User extends BaseEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ nullable: true })
  avatar: string;

  @Column({ nullable: false })
  nama: string;

  @Column({ unique: true, nullable: false })
  email: string;

  @Column({ nullable: true })
  password: string;

  @Column({ nullable: true })
  refresh_token: string;

  @Column({ nullable: true })
  role: string;

  @OneToMany(() => ResetPassword, (reset) => reset.user) // buat relasi one to many dengan tabel reset password
  reset_password: ResetPassword;

  @Column({ type: 'datetime', default: () => 'CURRENT_TIMESTAMP' })
  created_at: Date;

  @Column({ type: 'datetime', default: () => 'CURRENT_TIMESTAMP' })
  updated_at: Date;
}
```


*Langkah  8*

Buatlah method forgotPassword pada auth service

*auth.service.ts*

```tsx
import { HttpException, HttpStatus, Injectable } from '@nestjs/common';
...
import { MailService } from '../mail/mail.service'; // import mail service
import { ResetPassword } from './reset_password.entity'; // import reset password
import { randomBytes } from 'crypto'; // import cypto untuk membuat token dari random string

@Injectable()
export class AuthService extends BaseResponse {
  constructor(
    @InjectRepository(User) private readonly authRepository: Repository<User>,
    @InjectRepository(ResetPassword) private readonly resetPasswordRepository: Repository<ResetPassword>,  // inject repository reset password
    private jwtService: JwtService,
    private mailService: MailService,
  ) {
    super();
  }

  async forgotPassword(email: string): Promise<ResponseSuccess> {
    const user = await this.authRepository.findOne({
      where: {
        email: email,
      },
    });

    if (!user) {
      throw new HttpException(
        'Email tidak ditemukan',
        HttpStatus.UNPROCESSABLE_ENTITY,
      );
    }
    const token = randomBytes(32).toString('hex'); // membuat token
    const link = `http://localhost:5002/auth/reset-password/${user.id}/${token}`; //membuat link untuk reset password
    await this.mailService.sendForgotPassword({
      email: email,
      name: user.nama,
      link: link,
    });

    const payload = {
      user: {
        id: user.id,
      },
      token: token,
    };

    await this.resetPasswordRepository.save(payload); // menyimpan token dan id ke tabel reset password

    return this._success('Silahkan Cek Email');
  }
}
```


*Langkah 9*

Membuat endpoint reset password pada auth controller

```tsx
@Post('lupa-password')
  async forgotPassowrd(@Body('email') email: string) {
    console.log('email', email);
    return this.authService.forgotPassword(email);
  }
```


*Langkah 10*

Pengujin pada Postman


![Alt text](https://drive.google.com/uc?id=1IWxUPo_VG7obfycqXJbJvr7bL9iPTqR1)

Kemudian cek pada mailtrap apakan email berhasil masuk atau tidak

![Alt text](https://drive.google.com/uc?id=1O0fpFtWinWiNDN3pL0s2esTqkpZkrisq)