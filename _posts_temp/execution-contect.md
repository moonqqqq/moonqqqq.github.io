!! with out var keyword.

선언과 호출의 차이를 알아야 한다.

- execution context
- lexical environment
- scope chain
- closure
    - 클로저가 그래서 뭔데? 함수라는겨? 함수를 포함한 환경? 같은 이상한 소리 말고.
    - clousre 가 될수 있고 아닐수 있고 하다.
    - private 구현하기.

- 메모리? stack이라며 왜 안사라져 
    - heap
    - context 가 환경을 참조한다.

- 인터프리터와 실행컨텍스트 관계
    - 이 둘을 종합해서 돌아가는 순서

파싱 -> 실행 컨텍스트 생성 -> 코드 실행

TDZ는 선언은 됐지만 초기화는 안됐을 때를 나타낸다.

var는 "undefined" 로 초기화가 된다.

역시나 실행 컨텍스트는 모두 선언/실행과 관련이 있다.

- TDZ는 무슨 관련?


creation phase,
    - create VO
    - set scope
    - set this
excution phase

예전에는 VO라는 말로 정의했지만 ES6부터는 Lexical Environment라는 말을 쓴다.

Environment Record and Reference of outer Lexical Environment.

함수선언, 블록, catch, try를 만나면 새로 만든다.

 A Lexical Environment holds identifier-variable mappings.

##### 중요중요
# 변수나 함수를 위로 올린다는 개념보다 실행 컨텍스트에 미리 등록한다는게 맞지?

# The browser's JavaScript engine then creates a special environment to handle the transformation and execution of this JavaScript code. This environment is known as the Execution Context.
# The Execution Context contains the code that's currently running, and everything that aids in its execution.

자바스크립트 코드는 컴퓨터에서 실행되기 위한 무언가를 만든다. 먼저 컴퓨터 친화적인 코드로 변경되어야한다. 그리고 그 변경된 코드가 실행되어야 한다. 이 때 필요한 것들이 Execution Context안에 모두 존재한다.

코드들은 parse 되고 변수, 함수는 메모리에 저장된다. 컴퓨터에서 실행될수 있는 byte code로 변환되고 실행된다.

Global Execution Context (GEC)
Function Execution Context (FEC)


AST는 코드의 구조와 구문을 나타내는 데 주로 사용되며, 변수와 함수의 실제 값이나 실행 컨텍스트는 포함하지 않습니다. AST와 실행 컨텍스트는 각각 다른 역할을 수행하며, 자바스크립트 엔진이 코드를 실행하는 데 필요한 정보를 서로 보완합니다.

"실행 컨텍스트는 코드 실행 중에 생성되며," 선언에 실행되지않음. 선언에는 렉시컬 환경이 생성됨?


This process of storing variables and function declaration in memory prior to the execution of the code is known as Hoisting. Since this is an important concept, we'll talk about it briefly before moving on to the next stage.

#### execution context에서는 변수를 선언해놓기만하고. 코드가 지나갈때마다 변수를 설정한다.


클로저가 생성될 때, 자바스크립트 엔진은 해당 클로저가 참조하는 변수들을 힙 메모리에 저장합니다. 이는 클로저가 해당 변수를 지속적으로 참조할 수 있도록 하기 위함입니다. 변수는 원래 스택 프레임에 저장되지만, 클로저가 참조할 경우 힙 메모리로 이동됩니다.

- https://www.freecodecamp.org/news/execution-context-how-javascript-works-behind-the-scenes/
- https://dev.to/mihirverma7781/inside-javascript-engine-373
- https://mathiasbynens.be/notes/shapes-ics
- https://www.freecodecamp.org/news/javascript-under-the-hood-v8/
- https://medium.com/@yanguly/inside-javascript-engines-part-1-parsing-c519d75833d7