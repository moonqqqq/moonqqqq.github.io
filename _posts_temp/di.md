(SI팀과 신사업팀을 둘다 맡고 있다. 문제가 터지면 들어가는게 내 일이다보니 코드 베이스를 통일해둬야했다. 

여기서 중요했던게 헥사고날과 같은 많은 이들이 실제로 써보지않은 아키텍쳐는 쓰기가 힘들다. 아키텍쳐는 깊이 고민하지않고 이용하면 아무 의미가 없으니말이다. 그래서 우리는 아직까지도 layerd 아키텍쳐를 이용한다. nest.js에서 제공해주는 모듈단위의 개발이다. 거기에 repository 레이어까지만 추가한다.)

이래서 테이블데로 모듈 만들면 써큘러 디펜던시 생긴다.

모듈 묶어서

UserOrder, CustomerOrder 처럼.

관련있는 테이블을 하나의 모듈로 묶는다.o

모듈간에도 공통된 기능이 많이 존재한다. 보통 같은 부모 카테고리에서 뻗어나온 하위카테고리들이 그렇다.

파일이 많으면 관리가 힘들긴하지만, 파일을 많이 나눌수록 그나마 정리가 된다. 
하나의 모듈이 너무 큰 범위를 책임지면 안된다.

user 모듈이 company, customer API를 가지면 안된다.
신입일 때는 공통 모듈때문에 "user/company/:id", "user/customer/:id" 이렇게 했던 적도 있다.

customer
company             user        share




테이블 기준으로 repository를 나눌 필요는 없어보인다.


재사용성: B와 관련된 로직이 A 모듈 내에만 있으므로, 다른 모듈에서 재사용하기 어려울 수 있습니다.

1. GET Order/:id/receipt

장점:

	•	명확한 계층 구조: Order와 관련된 Receipt는 Order의 하위 자원으로 명확하게 표현됩니다.
	•	자연스러운 URL: Order와 관련된 자원을 계층적으로 접근할 수 있어 URL이 직관적입니다.
	•	API 명세 간결: 하나의 Order 컨트롤러 내에서 관리할 수 있어 관리가 편리합니다.

단점:

	•	단일 책임 원칙 위배 가능성: Order 모듈이 너무 많은 책임을 가지게 되어 복잡해질 수 있습니다.
	•	확장성 제한: Receipt가 Order 외 다른 자원과도 연관될 가능성이 있을 때 유연하지 않습니다.

2. GET receipt?orderId=123

장점:

	•	모듈화: Receipt를 독립된 모듈로 관리함으로써 단일 책임 원칙을 따를 수 있습니다.
	•	유연성: Receipt 모듈이 독립적이므로 다른 자원과 연관될 때도 쉽게 확장할 수 있습니다.
	•	재사용성: Receipt 관련 로직을 다른 곳에서도 재사용하기 쉽습니다.

단점:

	•	URL의 직관성 저하: URL에서 계층 구조가 명확하지 않아 직관성이 떨어질 수 있습니다.
	•	관리 복잡도 증가: 여러 모듈을 관리해야 하므로 초기 설정과 관리가 다소 복잡할 수 있습니다.

추천

기능적 확장성과 모듈화 측면에서 고려할 때, Receipt 모듈을 별도로 만드는 것을 추천드립니다. 이는 장기적으로 유지보수와 확장성에서 더 큰 이점을 가져올 것입니다. Receipt 모듈을 별도로 두면, 새로운 요구사항이나 기능이 추가될 때 더 유연하게 대처할 수 있습니다.

customers

orders

customerOrders
    const ids = customer.findMany(특정 조건);
    const orders = orders.findMany(ids)

모든게 user - order처럼 되지는 않는다. 하지만 보통 일대다 느낌일 때는 다가 일을 의존성으로 가지는게 높은 확률로 편하다.

bankAccount, transaction 간의 관계도 그렇다.

통장 목록을 가져올 때는 bankAccount 모듈을.
거래 내역을 가져올 때가 둘이 같이 쓰인다. 특정 조건에 맞는 bankAccount 들을 가져온뒤 그 통장들의 transaction을 검색하게 된다.


GET bankAccountTransactions API

const ids = bankAccount.findMany().map(each => each.id);
const transactions = bankAccountTransaction.findMany(ids);



- 모듈을 최대한 분리하는 사고 방식.  모듈은 테이블과 1대1로 매칭되는게 아니다.
Customer모듈, Order모듈 이 두가지의 모듈로 두 관련 데이터를 모두 책임지지않는다. 
CustomerOrder 모듈도 만들어서 모듈울 쪼갤수 있다. Customer 모듈과, order모듈은 각자 customer공통 기능(함수)를 가질수 있게 되고. order모듈 또한 CustomerModule, NonUserOrder(비회원 주문) 모듈을 만들어서 기능을 분리할수 있다. 하나의 모듈이 너무 여러 책임을 가지게 되면 기능이 너무 많아지다보니 유지보수가 어려워진다. 그리고 순환참조가 발생할 확률이 올라간다. 아래 그림을 보면 쉽게 이해될것이다.
이렇게 되면 customer 모듈과 order 모듈을 둘다 의존하게 된다. customerModule, NonUserModule, 이렇게 여러가지 모듈이 생기지만 순환참조는 없게 된다. customer, order모듈로 모든 기능을 만든다면 순환참조는 무조건 발생한다. 
- share 모듈
- 일대다 일때는 보통 다가 일을 의존하는게 좋다.
- 일대일 관계의 데이터들은 앞의 일에서 뒤의 일을 의존성으로 가지는게 좋다. 다만 현재 상황이 일대일일 지라도 나중에는 바귈수 있다. 내 경험을 공유해보자면. api가 직관적이어서 이렇게 했으나 나중에는 proof가 영수증말고도 다른 데이터들에도 들어갈일이 생겼다.
더 생각해보면 ReceiptProof, SomethingProof로 아에 따로 관리해도 되긴하다.
내 경험으로 생각 나는건. 영수증 처리에 대한 증거물 데이터를 작업할때였다. 증거물이 영수증데이터에만 들어가는 구조였어서 영수증에서 증거물을 의존하도록 만들었었다. A 도메인이 B 도메인과만 연관되어 있을 경우, A 모듈 내에서 B와의 관계를 관리하는 것이 좋습니다. 이는 API의 직관성과 관리의 용이성 측면에서 유리합니다. 하지만, 장기적으로 확장 가능성과 재사용성을 고려해야 한다면, B 도메인을 별도의 모듈로 분리하는 것도 고려해볼 수 있습니다.
A 도메인이 B 도메인과만 연관되어 있을 경우, A 모듈 내에서 B와의 관계를 관리하는 것이 좋습니다. 이는 API의 직관성과 관리의 용이성 측면에서 유리합니다. 하지만, 장기적으로 확장 가능성과 재사용성을 고려해야 한다면, B 도메인을 별도의 모듈로 분리하는 것도 고려해볼 수 있습니다.

1. GET Order/:id/receipt
2. GET receipt?orderId=123

1이 훨씬 보기 좋다. 하지만.

- lazy loading
    - lazy loading의 성능,
    - 전체 코드 디자인이 뒤죽박죽되버린다.
