근 1년간 소방수역할만 하다보니 우당탕 개발한 것밖에 기억이 안나는 상태다. 기존 코드에 맞춰서 할뿐이라 좋은 아키텍쳐를 고민하거나 OOP적인 코드를 짜거나 하는건 없었다. 오랜만에 처음부터 개발 해볼 겸 Backend를 개발하는 과정에 대해 차분히 정리하는 시간을 가져보려 한다. 그리고 회사에서 쓸 새로운 모듈도 만들겸.

1. request validation.
2. deserialize
3. change to object
4. process something
    - save on db
    - get from db
5. serialize


내 나름대로의 최선의 아키텍처 구조를 정리해보는 시간이다.

ddd, 헥사고날, 클린 등 여러 아키텍쳐가 있지만 외국인 개발자들이 계속해서 들어오고 나가고 하는 우리 회사에서는 그런것을 이용할수가 없다. 플젝이 위험해질때마다 내가 들어가야하는 상황이다보니 괜히 복잡한 아키텍처로 코드를 짜게 해버리면 나중에 들어갔을 때 이도저도 아닌 구조가 되어있는 상황이 너무 많았다. 그러다보니 쉬운 구조를 이용하도록 해야 나중에 내가 편했다. 그래서 기반으로 최종 결과물이 Nest.js에서 boilerplate로 제공해주는 구조에 repository, domain layer를 추가하는 것이다. 기존 MVC와 같다고 보면 된다.

그런데 아키텍처를 공부하면 할수록 트렌디한 아키텍쳐들이 그리 특별하지않다는 생각이 강해지는 중이다. OOP만 잘하면 다른 아키텍쳐들은 쉽게 쉽게 이해할수 있다고 믿는 사람이라 그럴수 있는데, 모든 아키텍쳐들은 높은 응집성, 낮은 결합성을 목표로 한다. 그것의 기초가 OOP지않나 싶다. 그래서 OOP적인 코드만은 무조건 넣어야했다. 그래서 Domain 모델을 추가해서 이용중이다. 

rich domain model 기반이다.

모든 것은 모듈로 관리된다.
개별 모듈은 같은 구조를 가진다.

module
    - view layer
        - http layer
        - socket layer
    - service layer
    - domains
    - repository layer

모든 비즈닛 로직은 service, domains 에 작성된다.
