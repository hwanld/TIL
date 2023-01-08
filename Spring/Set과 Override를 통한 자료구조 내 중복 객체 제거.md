#  Set과 Override를 통한 자료구조 내 중복 객체 제거
## 1. 문제 상황
<br>

```kotlin
class TravelFavorite(
    val region: String,
    val score: Int
) : Comparable<TravelFavorite> {
}

@Document(collection = "UsersTravelFavorite")
class UsersTravelFavorites(
    @Id
    val user_id: String,

    val travelFavorite_list: MutableList<TravelFavorite> = ArrayList<TravelFavorite>()

) {
    fun addTravelFavorite(travelFavorite: TravelFavorite) =
        travelFavorite_list.add(travelFavorite)
}

```
다음과 같이 MongoDB에 UsersTravelFavorites를 하나씩 저장하고자 하는 상황이었다. UsersTravelFavorites라는 객체를 하나 만들고, 객체 내부에 TravelFavorite라는 클래스를 리스트 형태로 넣어서 병렬적으로 보여주고자 하였다.

의도에 따르면 UsersTravelFavorites.travelFavorite_list 필드는 중복되는 값을 가지면 안되는 값이었는데, (사용자의 여행지 별 평가 점수를 저장하기 위함) 중복되는 값을 넣어도 계속해서 중복되는 값이 저장되는 것을 확인할 수 있었다.

따라서 ArrayList가 아닌 TreeSet을 사용해서 Set으로 바꾸고자 하였다.

```kotlin
class TravelFavorite(
    val region: String,
    val score: Int
) : Comparable<TravelFavorite> {
    
    override fun compareTo(other: TravelFavorite): Int {
        if (this.region > other.region) return 1
        return -1
    }
}

@Document(collection = "UsersTravelFavorite")
class UsersTravelFavorites(
    @Id
    val user_id: String,

    val travelFavorite_set: MutableSet<TravelFavorite> = TreeSet<TravelFavorite>()

) {
    fun addTravelFavorite(travelFavorite: TravelFavorite) =
        travelFavorite_set.add(travelFavorite)
}
```
다음과 같이 TreeSet을 사용할 수 있도록 변경하고, TreeSet의 경우 정렬된 Set이라는 특징을 가지고 있기 때문에 `Comparable<T>`를 상속받도록 구현한 후 `compareTo` 함수를 `override`하여 다시 `implementation`을 진행하였으나, 그럼에도 계속해서 중복되는 값이 나오는 것을 확인할 수 있었다.

## 2. 원인
Set 자료구조의 가장 큰 장점 중 하나는 `중복되는 원소를 허용하지 않음` 이라는 점이다. 따라서 종류 등 중복되지 않는 원소들을 사용하기 위해서는 Set 자료구조를 사용하는 것이 마땅하다. 그 중에서도 TreeSet을 사용하면 Set에 저장되는 자료들을 지정한 규칙 (Comparable, Comparator 등)에 의해서 정렬해서 저장까지 가능하다.

이제껏 Set 자료구조에 중복을 허용하지 않으면서 저장해야 하는 자료형들은 대부분 미리 구현되어 있는 자료형들이었다. (String, Int, Long 등) 하지만 현재 상황에서 Set 자료구조에는 내가 새롭게 정의한 `TravelFavorite` 라는 객체를 저장하고 있다.

여기서 원인을 찾을 수 있는데, 아래의 코드를 참고해보자.
```kotlin
class Subject (val name : String, val score : Float)

fun main(args: Array<String>) {
    
    val a = Subject ("자료구조", 4.5)
    val b = Subject ("소프트웨어공학", 4.5)
    val c = Subject ("자료구조", 4.5)
}
```
위 코드에는 Subject 라는 클래스를 정의하고 Subject 형태의 변수를 총 3개 선언하였다. a와 b는 당연히 다른 변수인데, 그럼 a와 c는 같은 변수일까? 같지만 다른 변수라고 말할 수 있을 것 같다.
> 물리적으로는 다른 객체이고 (메모리 참조 값이 다름), 논리적으로는 같은 객체이다 (클래스의 두 필드의 값이 같다)

처음 OOP를 배울 때 사용하는 개념인데, 클래스 두 개를 만들고 두 클래스에 같은 변수를 집어 넣는다고 해도 두 객체는 물리적으로 다른 객체이다. 두 객체가 같은 메모리 공간에 저장되는 것이 아니기 때문에, `AssertJ`와 같은 라이브러리를 통해서 테스트를 진행하더라도 두 객체는 다른 객체라고 나온다.

## 3. 해결
이러한 문제 때문에 논리적인 두 객체의 비교를 위해서 모든 객체는 `Objects.equals` 메소드를 상속받고 있다. 물론 내용은 `super.equals`이지만 `override`를 통해서 사용자가 원하는 대로 `implementation`이 가능하다. `hashcode` 메소드 역시 마찬가지이다.

Set에서 두 객채를 비교할 때 `equals`와 `hashcode` 메소드를 사용하기 때문에, 두 메소드를 각자 물리적인게 아닌, 논리적인 내용으로 새롭게 작성하면 Set이 원하는 대로 작동하도록 바꿀 수 있다.

```kotlin
package com.entrip.domain.entity

import org.springframework.data.annotation.Id
import org.springframework.data.mongodb.core.mapping.Document
import java.util.*


class TravelFavorite(
    val region: String,
    val score: Int
) : Comparable<TravelFavorite> {
    override fun compareTo(other: TravelFavorite): Int {
        if (this.region > other.region) return 1
        return -1
    }

    override fun equals(other: Any?): Boolean {
        if (other !is TravelFavorite) return false
        if (other.region == this.region && other.score == this.score) return true
        return false
    }

    override fun hashCode(): Int {
        return Objects.hash(region, score)
    }
}

@Document(collection = "UsersTravelFavorite")
class UsersTravelFavorites(
    @Id
    val user_id: String,

    val travelFavorite_set: MutableSet<TravelFavorite> = TreeSet<TravelFavorite>()

) {
    fun addTravelFavorite(travelFavorite: TravelFavorite) =
        travelFavorite_set.add(travelFavorite)

}
```
추가로 `equals` 메소드의 전달 인자는 Any? 이다. 즉 아무 객체와 비교하더라도 해당 메소드가 정상적으로 동작해아 한다. 따라서 우선적으로 other로 주어진 객체가 비교하고자 하는 대상의 객체와 타입이 같은지부터 확인해야 하는데, 여기서 Kotlin의 `Smart Casting`을 사용해서 `is` 를 사용한다면 쉽게 해결할 수 있다. (Java는 instanceof)
