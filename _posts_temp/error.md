에러라는게 보통은 서비스 레이어에서 발생한다.

- Repository
repository 레이어에서 발생하는건 다 문법적인 문제여야하는거고.
404는 null로 반환해야할까? 아니면 에러로 반환해야할까?

- Service
레이어에서는 비지니스 로직 관련된 에러가 다 발생할수 있는거고.

- Controller는....
굳이 에러가 발생해야하나 싶다.
에러가 발생하는 케이스가 보통 A Service에서 기존 데이터를 찾아서 리턴, 찾은 데이터를 가지고 B Service에서 비지니스 로직을 처리한다.

A Service에서 데이터를 못찾았을 떼 에러를 발생시키면.

A Service layer를 이용하는 모든 컨트롤러에서는 catch() 구문이 들어가야한다.

# 이건 좋은듯하다. FindOrThrow(), FindOrNull() 이렇게 분리해서 써야하는건가? !!

# 에러 메시지 constant로 만들어 쓰는게 좋은거같은데 지금까지 쓰던 에러 메시지 constant는 너무 크다.

const CODE1 = {
  USER_ALREADY_EXISTS: 'USER_ALREADY_EXISTS',
  INTERNAL_SERVER_ERROR: 'INTERNAL_SERVER_ERROR',
} as const;

export interface ICode1ErrorContent1 {
  errorCode: keyof typeof ALL_CODE;
  message: string;
}

export const CODE1_ERROR_BODY: Record<keyof typeof CODE1, ICode1ErrorContent1> = {
  USER_ALREADY_EXISTS: {
    errorCode: CODE1.INTERNAL_SERVER_ERROR,
    message: 'User already exists',
  },
  INTERNAL_SERVER_ERROR: {
    errorCode: CODE1.INTERNAL_SERVER_ERROR,
    message: 'Internal server error',
  },
}

const CODE2 = {
  NO_CERTIFICATE: 'NO_CERTIFICATE',
  REQUIRED_CERTIFICATE_NOT_EXISTS: 'REQUIRED_CERTIFICATE_NOT_EXISTS',
} as const;


이렇게 분리하면 좋을듯하다.