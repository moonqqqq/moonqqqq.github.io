# Nested Loop Join
# Hash Join
# Sort Merge Join

- 개별 조인들을 사용해야하는 때, 사용하면 안되는 때
- 개별 조인들 설명


----


# 랜덤 액세스 vs 순차 액세스
조인 종류를 보기전에 먼저 랜덤 액세스, 순차 액세스가 뭔지 알아야한다.

## 랜덤 액세스
랜덤 액세스는 특정 행을 하나 찝어서 찾는 것이다. 
"랜덤" 이라는 단어가 붙어서 이름과 내용이 잘매칭되지않는다. 데이터가 저장된 순서에 상관없이 접근한다고 해서 "랜덤"이 붙었다고 한다. (난 아직도 이름을 잘못지었다고 생각한다..) 그리고 메모리의 임의 위치에 저장된 데이터를 접근할 수 있음을 의미한다고 여러곳에서 설명한다.
데이터베이스는 특정 행을 찾는 과정에서 "tree-search"가 실행되는데 이 과정이 여러번 진행되면 순차 액세스에 비해 훨씬 시간이 많이 든다. 예를 들어 <U>(캐시, 페이지 프리페칭이 없다고 가정하자.)</U> ID가 1,2,3 인 데이터를 가져와야할 때 랜덤액세스를 이용하게 되면 1,2,3 ID에 대한 랜덤엑세스가 3번 진행된다. (tree-search가 3번 진행된다.)

## 순차 액세스
반면에 순차 액세스는 디비가 데이터를 검색할 때 특정 행을 가져오는것이 아니라 한 뭉치의 데이터들을 가져오는 것을 활용한다. 디비에서 데이터는 페이지로 관리되고 페이지에는 순서대로 여러개의 데이터가 저장되어있다. ID가 1,2,3인 데이터가 같은 페이지에 저장돼있고, ID가 4,5,6인 데이터들이 같은 페이지에 저장돼있다고 보면 된다. 그렇기 때문에 ID === 1 인 데이터를 가져온다면 이미 2,3 데이터도 가져온것이기 때문에 ID === 1 데이터를 읽은 후 바로 **추가적인 tree-search없이** ID === 2 인 데이터에 접근할수 있다.


이제 join algorithm 종류를 알아보자.

## Nested Loop Join
이름 그대로 중첩 loop 방식으로 조건에 맞는 데이터를 찾는다. 중첩 for문이라고 생각하면 된다.
첫번째 테이블에서 데이터를 하나 선택하고 두번째 테이블을 선회하면서 둘의 조인키를 확인하며 조건이 맞는지 확인한다.
User와 School 테이블이 있고 userId로 조인된 상태라면 아래 처럼 작동한다.

```ts
for (const user of users) { // 첫번째 Loop
    for (const school of schools) { // 두번째 Loop
        if (user.schoolId === school.id) { // 두 테이블의 데이터를 가져왔으니 조인 조건 확인.
            pushToResultArray(user, school) // 조건이 맞다면 결과에 추가.
        }
    }
}
```
이렇게 하면 복잡도가 n제곱이라서 성능상 좋지않은거냐고 할수 있다. 맞다. 그래서 많은 양의 데이터를 검색할때는 성능이 좋지않다.
그런데 대부분의 쿼리에는 **where절**이 있기때문에 최대한 loop에 돌아가는 범위를 줄이게 된다. where로 데이터 범위를 줄일수 없다면 데이터베이스가 알아서 다른 join 알고리즘으로 변경하여 성능을 최적화한다.


## Hash Join
Hash Join은 두 테이블이 조인하는 특정 값을 이용한다. 둘이 같은 값을 가지고 있기 때문에(user, school이라면 user가 schoolId를 가지므로 같은 값을 가짐.) 같은 해시 함수를 실행하면 같은 응답값이 나오는 걸 이용하는 것이다.

먼저 해시 테이블을 만든다. 조인된 두 테이블 중 사이즈가 작은 테이블을 찾는다. 그리고 작은 사이즈의 테이블을 순회하며 해시테이블을 만든다. <U>해시 테이블</U>의 키는 조인에 이용되는 값을 해시 함수 실행시켜 얻은 값이다. <U>해시 테이블의 값</U>은 해당 열이다. 해당 열의 일부분이 저장될수도 있고 전부가 저장될수도 있다. 혹은 해당 열의 메모리 주소값일수도 있다.(메모리 주소로 값을 저장하면 추후에 메모리 주소로 다시 접근하여 데이터를 얻어야하는 단점이 있다.)

데이터가 아래와 같다면

![1](/img/writing-images/usertable.png)
![2](/img/writing-images/schooltable.png)

먼저 School테이블이 사이즈가 더 작기때문에 School 테이블을 이용하여 해시 테이블을 만든다. User와 School 테이블은 schoolId로 서로 조인하고 있기 때문에 School.id를 해시함수 실행시켜 키 값을 설정한다. 그리고 해시 테이블의 값으로는 해당 열 데이터의 일부를 저장한다. 예시에서는 school 데이터의 ID를 저장한다.

![2](/img/writing-images/hashtable.png)

이제 나머지 테이블(크기가 좀더 큰 테이블: User)에서 작업을 시작한다. User 테이블을 순회하여 School과의 조인을 나타내는 값을 해시 함수 실행시킨다. 해시 함수 실행 결과 값이 해시테이블에 존재한다면 쿼리에 현재 데이터 열과 값이 같은 해시 테이블의 데이터를 결합시킨다. 그리고 결과 목록에 추가한다. User 테이블에서 순회를 마치면 결과 목록이 완성된다.

나머지 테이블의 데이터를 하나하나 순회하면서 join에 사용된 값을 해시함수로 실행시켜 해시테이블의 매칭되는 값이 있는지 확인한다. 매칭되는 값이 있다면 그 둘 데이터를 결합하여 결과 목록에 추가한다.

대략 코드로 표현하면 이렇다.
```ts
const smallerTable = findSmallerTable(Users, Schools);
const biggerTable = Users === smallerTable ? Schools : Users;

const hashTable = createHashTable(smallerTable);

for(const eachRow of biggerTable) {
    const joinedTable = findMatchingDataFromOtherTable(hashFunc(eachRow.joinKey))

    return pushToJoinResult(joinedTable, eachRwo)
}
```

## Merge Join




# 상황 별 장단점.

## 인덱스와 조인 쿼리.

Non clustered Index를 이용할 때는 hash join의 성능이 발휘되지않는다. 
Hash Join은 순차 액세스를 하는데 non clustered 인덱스로 검색하게 되면 순차 액세스가 불가하다.

## 어느상황에 써야하냐.

사실 join 쿼리 전략은 디비 자체적으로 대부분 최적화되있어 우리가 건드릴 일이 크지 않다.
역시나 인덱스를 제대로 안갈어두는게 대부분의 문제이다.




흐름을 코드로 파악하면 이렇다.
```ts
function hashJoin(Users, Schools) {
    const smallerTable = findSmallerTable(Users, Schools);
    const biggerTable = Users === smallerTable ? Schools : Users;

    const hashTable = createHashTable(smallerTable);

    for(const eachRow of biggerTable) {
        const joinedTable = findMatchingDataFromOtherTable(hashFunc(eachRow.joinKey))

        return pushToJoinResult(joinedTable, eachRwo)
    }
}
```