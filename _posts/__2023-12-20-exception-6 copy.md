---
title: Nest.js 구조화/계층화된 Exception 관리.
description: Nest.js layered exception, error
header: Nest.js 구조화/계층화된 Exception 관리.
tags:
 - exception
 - node.js
---


##### 목차
- [Exception의 내용](#내용)
- [계층화/구조화](#발생)
  - [Layer2](#layer2)
  - [Layer3](#layer3)
  - [Layer4](#layer4)
- [발생시킨 Exception 처리 - 익셉션 필터 구조화](#필터)
  - [계층별](#eachLayer)
  - [Third party library](#thrid)
  - [Unhandled exception](#unhandled)

------


> 지금 이용하지않는 버전이므로 참고하지않는게 좋다.

## 1. Exception의 내용 <a id="내용"></a> 
구조화에 들어가기전에 잠시 정리하고 넘어가면. <br>
Exception은 클라이언트에게 API 이용 가이드가 되어야한다.
그러기 위해 형식이 **고정**돼어야 한다. 디테일해야 하며, 클라이언트가 이용할수 있는 **단서**를 주는 것도 좋다.

내가 이용하는 Error 내용물은 아래와 같다.
```typescript
// error-body.ts

// 익셉션 바디의 타입.
export interface IErrorContent {
  message: string;
  errorCode: string;
}

export const ErrorCode = {
  POST_INPUT_WRONG: 'POST_INPUT_WRONG',
  ALREADY_WITHDRAWED_USER: 'ALREADY_WITHDRAWED_USER',
  CANNOT_FOLLOW_SELF: 'CANNOT_FOLLOW_SELF',
  CHATROOM_DOES_NOT_EXIST: 'CHATROOM_DOES_NOT_EXIST',
  INTERNAL_SERVER_ERROR: 'INTERNER_SERVER_ERROR',
} as const;

// 대부분의 경우 errorCode와 message의 내용이 텍스트가 같긴해서 message의 효용성을 의심할수 있지만
// 다를 때가 꽤 있기때문에 필요하다.
export const ErrorBody: Record<keyof typeof ErrorCode, IErrorContent> = {
  POST_MINIMUN_LENGTH: {
    errorCode: ErrorCode.POST_INPUT_WRONG,
    message: 'Post minimum length is 10',
  },
  ALREADY_WITHDRAWED_USER: {
    errorCode: ErrorCode.ALREADY_WITHDRAWED_USER,
    message: 'User already exists',
  },
  CANNOT_FOLLOW_SELF: {
    errorCode: ErrorCodes.CANNOT_FOLLOW_SELF,
    message: 'Cannot follow self',
  },
  CHATROOM_DOES_NOT_EXIST: {
    errorCode: ErrorCodes.CHATROOM_DOES_NOT_EXIST,
    message: 'Chatroom does not exist',
  },
  INTERNAL_SERVER_ERROR: {
    errorCode: ErrorCode.INTERNAL_SERVER_ERROR,
    message: 'Internal server error',
  },
};

export type TErrorBody = (typeof ErrorBody)[keyof typeof ErrorBody];
```
<br>
사용할 때는 아래와 같이.
```typescript
if (followingId === my.id) {
    throw new CannotFollowSelf(API_ERROR.CANNOT_FOLLOW_SELF);
}
```

<br>
errorStatus(ex. 401, 409)와는 다른 errorCode라는 데이터를 가지고 있다. errorCode는 errorStatus가 나타내는 큰 카테고리안에서 좀더 디테일한 카테고리화를 가능하게 해준다. **그리고 클라이언트가 같은 status안에서도 분기처리를 하고 싶은 상황이라면 errorCode를 이용해서 분기처리를 할수 있게 된다.**

아래 코드는 백엔드에서 이용할 Exception이 상속받게 될 BaseException이다.
위 코드에서 만든 **TErrorBody를 이용하여 Exception의 메시지 형식을 강제하고 있다.**

```typescript
import { TErrorBody } from '../constants/error-body';

export abstract class BaseException extends Error {
  errorCode: string;

  constructor(errorBody: TErrorBody) { // TErrorBody 이용
    super(errorBody.message);
    this.errorCode = errorBody.errorCode;
  }
}
```

<br>

## 2. 계층화/구조화 <a id="발생"></a>
**Exception을 계층화한다.**
Nest.js에서 기본으로 제공해주는 HttpException이 있지만 이름 그대로 Http를 위한 것이다. HTTP는 일반적으로 controller단이다. service, repository 계층에서도 이를 이용한다면 하나의 기능이 너무 커져버리게 된다. 작은 프로그램에선 써도 되겠지만 프로그램이 커질수록 너무 복잡해질수 있다.(통일된 Exception 구조가 없다면 HttpException만 쓰는 것도 괜찮긴하다.)

계층화의 구조는 아래와 같다.

```
BaseException: 베이스
  - LayerException: 개별 프로그램 폴더 구조에 따라 만들어진다.
    - CategoryException: throw할 익셉션의 카테고리화를 위한 용도.
      - DetailedException: throw할 익셉션.
```

<br>

Exception 구조화는 당연히 기본 Error 객체를 상속받는 것으로부터 시작된다.

```ts
import { TErrorBody } from '../constants/error-body';

export abstract class BaseException extends Error {
  errorCode: string;

  constructor(errorBody: TErrorBody) {
    super(errorBody.message);
    this.errorCode = errorBody.errorCode;
  }
}
```
<br>
 그 후 정의한 BaseException을 확장해서 계층화한다. 계층화는 아래처럼 된다.

```typescript
// layer1
import { BaseException } from './base.exception';

// layer2
export abstract class ServiceLayerException extends BaseException {}
export abstract class RepositoryLayerException extends BaseException {}

// layer3
// Do not makes it have the same name with nest.js built-in exception name
export abstract class AuthenticationException extends ServiceLayerException {}
export abstract class AuthorizationException extends ServiceLayerException {}
export abstract class WrongInputException extends ServiceLayerException {}

// layer4
export class CannotFollowSelf extends WrongInputException {}
```

###### Layer 2<a id="layer2"></a> 
코드의 계층만큼 BaseException을 상속받는 Exception이 생겨난다. ControllerException은 따로 없다. HttpException을 바로 이용한다.

###### Layer 3<a id="layer3"></a> 
계층마다 이용할 Exception 내부에서 Exception의 카테고리별로 나뉘게 된다. AuthenticationException, WrongInputException...

###### Layer 4<a id="layer4"></a> 
코드단에서 직접 throw할 Exception이다. Exception의 이름이 꽤나 디테일해진걸 볼수 있다. 그리고 다른 Exception들과 달리 "layer4"만 **추상 클래스**가 아닌걸 확인할수 있다.
```typescript
// Throw 되는 layer 4 exception
if (followingId === my.id) {
    throw new CannotFollowSelf(API_ERROR.CANNOT_FOLLOW_SELF);
}
```

<br>

## 3. 발생시킨 Exception 처리 - 익셉션 필터 구조화<a id="필터"></a>

###### 계층별<a id="eachLayer"></a> 
익셉션 계층화의 제일 큰 목적이 이 부분이라고 생각된다. 계층을 나눈 뒤에 한 군데에서 받으면 의미가 없어진다. (분기를 잘 처리해두면 문제가 없겠지만 많은 분기는 결국엔 부작용을 만든다.) 계층화해둔 익셉션을 layer2(service, repository)에 따라 개별 filter로 받아서 처리한다. (filter의 할일은 결국엔 발생한 에러를 서버내에서 잘 처리하고 클라이언트에게 Http Response로 변환해 전달해주는 것이다.)

```
src
  filters
    http-exception.filter.ts
    service-layer-exception.filter.ts
    repository-layer-exception.filter.ts
    prisma-exception.filter.ts   //  !!
    unhandled-exception.filter.ts   // !!
```
<br>
개별 layer별로 필터를 구성해두고, 익셉션도 이미 구조화된 상태라면 처리하는 일은 너무나 쉽다. 
1. "@Catch()" 의 인자로 Layer2 Exception을 입력한다. (Layer2를 상속받은 모든 Exception을 catch할수 있게 된다.)
2. Layer3(카테고리별)에 따라 HttpStatus를 매칭해준다.
3. 그런 뒤에 Http 응답으로 만들어 res로 전달한다.

아래는 서비스 계층을 위한 필터 예시코드이다.

```typescript
// Service Layer Exception Filter
import { ArgumentsHost, Catch, HttpStatus } from '@nestjs/common';
import { Response } from 'express';
import {
  AuthenticationException,
  AuthorizationException,
  ServiceLayerException,
} from '../../shared/exceptions/service-layer.exception';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch(ServiceLayerException) // 1. ServiceLayerException만 처리.
export class ServiceLayerExceptionToHttpExceptionFilter extends BaseExceptionFilter {
  catch(exception: ServiceLayerException, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();

    // 2. layer3에 따라 status를 매칭해준다.
    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    if (exception instanceof AuthenticationException)
      status = HttpStatus.UNAUTHORIZED;
    if (exception instanceof AuthorizationException)
      status = HttpStatus.UNAUTHORIZED;

    // 3. Http 응답으로 전달
    res.status(status).json({
      errorCode: exception.errorCode,
      message: exception.message,
    });
  }
}
```
<br>
###### Thrid part library exception<a id="thrid"></a> 

그리고 계층에 포함시키기 애매한 exception들이 있다. 라이브러리 자체 익셉션들이 그렇다. 이러한 익셉션들은 따로 필터를 만들어 처리하면 코드가 깔끔해진다. 아래 "prisma-exception.filter.ts 참고."

```typescript
// Prisma Exception Filter
import { ArgumentsHost, Catch, HttpStatus } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';
import { Prisma } from '@prisma/client';
import { Response } from 'express';
import { PrismaErrorCodes } from '../../shared/database/prisma/prisma.constant';

@Catch(Prisma.PrismaClientKnownRequestError)
export class PrismaClientExceptionFilter extends BaseExceptionFilter {
  catch(exception: Prisma.PrismaClientKnownRequestError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const message = exception.message.replace(/\n/g, '');

    if (exception.code === PrismaErrorCodes.UNIQUE_INDEX_CONFLICT) {
      const status = HttpStatus.CONFLICT;
      response.status(status).json({
        errorCode: message,
        message: exception.message,
      });

      return;
    }
  }
}
```
<br>
###### Unhandled Exception<a id="unhandled"></a> 
마지막으로 중요한 예상하지 못한 Error에 대한 처리가 필요하다.
@Catch() 데코레이터의 인자를 공백으로 두면 모든 에러에 대해 처리할수 있게 된다.
이 필터에서는 개발자에게 알림을 가도록 어떠한 연결이던지 해두는 곳이다. slack, sms etc.
```typescript
// unhandled filter exception
import { Catch, ArgumentsHost, HttpStatus } from '@nestjs/common';
import { Request, Response } from 'express';
import { ILoggerService } from '../../shared/logger/interface/logger-service.interface';
import { ErrorBody } from '../../shared/constants/error-body';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class UnhandledExceptionFilter extends BaseExceptionFilter {
  constructor(private readonly logger: ILoggerService) {
    super();
  }

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const req = ctx.getRequest<Request>();

    const status = HttpStatus.INTERNAL_SERVER_ERROR;
    const message =
      process.env.NODE_ENV !== 'prod' ? (exception as any).message : '';
    const error = ErrorBody.INTERNAL_SERVER_ERROR;

    // Send alert here or in logging server
    this.logger.error(
      `[${new Date()}] [${req.method}] ${req.url} / body: ${
        req.body
      } / code:${error}- ${exception} - ${(exception as any).stack}}`,
    );

    res.status(status).json({ ...error, detail: message });
  }
}
```

마지막으로 Filter를 구조화할 때 조심하지않으면 500으로 처리돼야할 것들이 다른 filter에서 처리되는 경우도 간혹 있다. Catch()에 넣는 인자를 잘 확인해볼 필요가 있다.