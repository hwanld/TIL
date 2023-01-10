#  SpringBoot Test Error : No Serializer found for class<br>

## 1. 문제
* Controller 계층에 대한 테스트를 작성하던 중, 다음과 같은 에러를 마주할 수 있었다.
> com.fasterxml.jackson.databind.exc.InvalidDefinitionException: No serializer found for class com.ReRollBag.domain.dto.UsersSaveRequestDto and no properties discovered to create BeanSerializer (to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS)

* Test Code 의 경우 직접 작성하면서 서비스에 적용해 보는 것이 처음이었기 때문에, 해당 에러가 왜 발생하는지 이해할 수 없었다만, Serializer 와 관련이 있다면 분명 JSON 과 관련이 있을 것으로 생각되었다.
* 에러가 발생한 부분의 코드는 아래 코드에서 ObjectMapper 부분이었다.
```java
        mockMvc.perform(post("/api/users/save")
                .contentType(MediaType.APPLICATION_JSON)
                .content(new ObjectMapper().writeValueAsString(requestDto)))
                .andExpect(status().isOk())
                .andExpect(content().json(new ObjectMapper().writeValueAsString(responseDto)))
                .andDo(print());
```

## 2. 원인
* ObjectMapper 의 경우 Json 에서 Java 객체로, Java 객체에서 Json 으로의 변환을 편하게 도와주는 객체이다. Jackson 을 기반으로 동작하는 객체이다.
* 그런데 ObjectMapper 를 통해서 Json 으로 변환하는 과정에서 변환하고자 하는 객체의 필드의 값을 알 수 없다면 Jackson 은 객체의 값을 알아낼 수가 없다.

## 3. 해결
* Jackson 이 객체를 잘 읽어올 수 있도록 변환하는 객체들에 `@Getter` 어노테이션 하나를 사용하면 간단하게 해결할 수 있었다.
```java
package com.ReRollBag.domain.dto;

import com.ReRollBag.domain.entity.Users;
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public class UsersSaveRequestDto {
    private String usersId;
    private String nickname;
    private String password;

    public Users toEntity() {
        return Users.builder()
                .usersId(usersId)
                .nickname(nickname)
                .password(password)
                .build();
    }
}

```


