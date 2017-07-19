---
layout: post
title:  "${} 이거 나만 안되?? Javascript template string"
---
JavaScript 예제를 따라하다
```
> let str = 'hello';
> console.log(`${str} world');
> hello world
```
위와 같이 ${} 문법을 본적이 있다.

분명 똑같이 코딩을 했는데 왜...
```
> '${str} world'
```
이런 식으로 출력이 되냐고...!!!

한참 삽질하다 알아낸 반전은
''이게 일반적인 홑 따옴표가 아닌
``back quote 라는 것이다.

즉 키보드 숫자열 가장 왼쪽에 있는 back quote를 사용해야만 template string을 사용할 수 있다.

그것도 모르고.. es6로 컴파일 하려면 특별한 옵션이 있는지 아니면 내 설정이 잘못 된건지.. 거의 1시간 정도 삽질을 했다.