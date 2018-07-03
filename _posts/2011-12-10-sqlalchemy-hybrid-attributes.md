---
layout: entry
title: SQLAlchemy의 Hybrid Attributes를 이용해서 MySQL에서의 위치정보를 깔끔하게 암호화하기
author: 김재석
author-email: shinvee@spoqa.com
description: 위치 기반 서비스(Location-based Service)를 국내에서 합법적으로 운영하기 위해선 등록위치정보의 저장을 전부 암호화해야 합니다. 저희는 SQLAlchemy의 Hybrid Attributes를 사용하여 위치정보를 암호화하였습니다.
---

**[위치 기반 서비스][](Location-based Service)**를 국내에서 합법적으로 운영하기 위해선 위치기반 사업자 등록이 필요하다. 이 과정은 여러가지 보안사항에 대한 적용을 강제하고 있는데, 이 중 위치정보의 저장을 전부 암호화해야 한다는 조항이 있다.

이미 서비스를 어느정도 구현하여 운영중이라면 특히 이 조항이 매우 귀찮거나 부담되는 일일 수가 있다. 모든 위치정보 기록시점에서 암호화가 되어 저장되어야하고, 위치정보를 참조하거나 계산하는 시점에서 바로 복호화가 되어야 하기 때문이다. 

스포카 개발팀은 이미 여러 곳에서 위치 정보를 암호화하지 않은 형태로 저장하고 불러오고 있었기 때문에, 기존 코드를 거의 유지하면서 위치정보 암호화를 투명하게 구현하기 위해 [SQLAlchemy]의 [Hybrid Attributes]를 사용하였다.

## SQLAlchemy?
---

[SQLAlchemy]는 파이썬에서 이용할 수 있는 [ORM]으로, 상당히 고 수준의 인터페이스를 제공하여 완벽에 가까운 수준의 Object-Relation Mapping을 구현할 수 있게 도와준다.

## Hybrid Attributes?
---

최근 0.7에선 [Hybrid Attributes]라는 것이 추가되었는데, 이 기능은 클래스가 정의한 속성이 사용 시점에 따라 파이썬 네이티브로, 혹은 같은 표현식의 SQL Query로도 이용할 수 있게 도와준다. 

일반적인 클래스의 속성형이라면 이미 파이썬으로 가져온 엔티티에 한해서 이용할 수밖에 없으나, [Hybrid Attributes]를 이용하면 select query의 조건절같이 SQL 문법으로 변환되어야 하는 시점에서도 해당 속성 요소를 활용할 수 있다.

## 방법
---

먼저 기존의 서비스는 엔티티에 위치 값을 저장시키기 위해 **LocationMixIn**이라는 클래스를 정의해놓고 필요한 엔티티 클래스가 이를 상속받게 하였다.

<script src="https://gist.github.com/1452137.js?file=gistfile1.py">  </script>

우리는 이를 실제로는 암호화해서 저장한 후, 참조하는 모든 환경에서 다시 원래 값으로 복호화하는 hybrid property를 구현하고 이로  구성된 새로운 클래스로 기존 LocationMixIn을 교체할 것이다.

### 1. MySQL의 AES 암호화 알고리즘과 호환되는 암호화 알고리즘 구현
---
MySQL에선 AES 압축 알고리즘이 기본으로 제공되며 이는 각각 AES_ENCRYPT, AES_DECRYPT이다. 해당 함수들이 알고리즘을 설명한 [스펙문서](http://dev.mysql.com/doc/refman/5.5/en/encryption-functions.html#function_aes-encrypt)에 따라 python에서 구현해주면 된다.

<script src="https://gist.github.com/1452269.js?file=gistfile1.py">  </script>

### 2. 기존 LocationMixIn을 대체할 새로운 클래스 정의
---
준비가 되었다면 이제 새로운 **EncryptedLocationMixIn**을 만들어보자. 아래와 같이 암호화된 값을 받을 공간을 할당받게 먼저 구성한다.

<script src="https://gist.github.com/1452328.js?file=gistfile1.py">  </script>

하나의 위치 정보 속성에 대한 hybrid property를 구현하기 위해 세가지 함수를 정의한다. 

<script src="https://gist.github.com/1452349.js?file=gistfile1.py">  </script>

lat 이라는 함수가 암호화된 lat을 복호화해서 반환하게 구현한 후 이를 **@hybrid_property**로 만든다. 

<script src="https://gist.github.com/1452370.js?file=gistfile1.py">  </script>

기본 수식 정도는 hybrid_property 선언만으로 자연스럽게 SQL 표현식으로 변환이 되지만, 우리는 지원하지 않는 MySQL built-in function을 이용하고 있기 때문에 위와 같이 복호화에 대한 SQL expression을 새로 구현해준다. [**sqlalchemy.sql.expression.func**](http://www.sqlalchemy.org/docs/core/expression_api.html#sqlalchemy.sql.expression.func)는 SQL function을 표현할 수 있게 해주는 객체 인스턴스이다.

<script src="https://gist.github.com/1452406.js?file=gistfile1.py">  </script>

마지막으로 값이 저장될 때 암호화되어 저장될 수 있도록 한다. 16자로 자르는 이유는 위치 값이 그보다 길어질 때 암호화된 값이 잘려서 들어갈 위험을 방지하기 위해서이다.

**lng**도 마찬가지로 구현해주면 된다.

### 3. 기존의 LocationMixIn을 교체
---
현재 만든 **EncryptedLocationMixIn**은 MySQL의 built-in function을 이용하기 때문에 다른 DB에서는 작동하지 않을 것이다. 우리는 프로덕션 서비스에서만 암호화되어 작동되면 되므로 EncryptedLocationMixIn을 mysql 연결시에만 교체하도록 하였다.

<script src="https://gist.github.com/1452426.js?file=gistfile1.py">  </script>

이와 같이 하면 SQLite와 같은 DB에선 평범하게 값이 저장이 되고, MySQL에서만 암호화되어 저장되게 된다.

## 장점
---
hybrid property를 이용해서 위치정보를 암호화하면, 암호화된 위치정보를 어떤 상황에서든 암호화되지 않은 위치정보를 다루듯이 이용할 수 있다는 것이 장점이다.

<script src="https://gist.github.com/1452468.js?file=gistfile1.py">  </script>

이와 같이 가져온 객체의 위치값을 바로 복호화해서 받아내며,

<script src="https://gist.github.com/1452483.js?file=gistfile1.py">  </script>

이와 같이 숫자 계산과 비교를 하는 조건절에도 대응을 할 수 있다. 첫째 줄과 컴파일된 WHERE절을 비교해보자.

이러한 방식을 통해 [스포카]는 기존의 핸들러/모델 메서드들이 위치값에 접근하기 위해 구현된 코드를 건드리지 않으면서, 저장되는 위치정보를 성공적으로 암호화할 수 있었다.

  [위치 기반 서비스]: http://ko.wikipedia.org/wiki/%EC%9C%84%EC%B9%98_%EA%B8%B0%EB%B0%98_%EC%84%9C%EB%B9%84%EC%8A%A4
  [SQLAlchemy]: http://www.sqlalchemy.org/
  [ORM]: http://en.wikipedia.org/wiki/Object-relational_mapping
  [Hybrid Attributes]: http://www.sqlalchemy.org/docs/orm/extensions/hybrid.html
  [스포카]: http://www.spoqa.com/
