# OrderProjectWithJPA



[ #12- 상품 수정 폼 이동 ] 커에 이어지는 설명입니다.



## 변경 감지와 병합(merge) 

참고: 정말 중요한 내용이니 꼭! 완벽하게 이해하셔야 합니다. 

준영속 엔티티?

- 영속성 컨텍스트가 더는 관리하지 않는 엔티티를 말한다. 
  (여기서는 itemService.saveItem(book) 에서 수정을 시도하는 Book 객체다. Book 객체는 이미 DB 에 한번 저장되어서 식별자가 존재한다. 이렇게 임의로 만들어낸 엔티티도 기존 식별자를 가지고 있으면 준 영속 엔티티로 볼 수 있다.)



-  **준영속 엔티티를 수정하는 2가지 방법** 

  1. **변경 감지 기능 사용** 
  2. **병합( merge ) 사용** 

  

  

### 1. 변경 감지 기능 사용

영속성 컨텍스트에서 엔티티를 다시 조회한 후에 데이터를 수정하는 방법이다.

```java
@Transactional
void update(Item itemParam) {     //itemParam: 파리미터로 넘어온 준영속 상태의 엔티티
    Item findItem = em.find(Item.class, itemParam.getId());     //같은 엔티티를 조회한다.
    findItem.setPrice(itemParam.getPrice());    //데이터를 수정한다.
}
```

- 트랜잭션 안에서 엔티티를 다시 조회, 변경할 값 선택
  선택 이후에 트랜잭션 커밋 시점에 변경 감지(Dirty Checking) 동작해서 데이터베이스에 UPDATE SQL 실행



### 2. 병합 사용

병합은 준영속 상태의 엔티티를 영속 상태로 변경할 때 사용하는 기능이다.

```java
@Transactional
void update(Item itemParam) { //itemParam: 파리미터로 넘어온 준영속 상태의 엔티티
 Item mergeItem = em.merge(itemParam);
}
```



### 3. 병합: 기존에 있는 엔티티

![image](https://github.com/LimdaeIl/OrderProjectWithJPA/assets/131642334/8211a04e-3e47-4932-a49d-15a2ed104d7a)


**병합 동작 방식** 

1. `merge()` 를 실행한다.
2. 파라미터로 넘어온 준영속 엔티티의 식별자 값으로 1차 캐시에서 엔티티를 조회한다. 
   - 만약 1차 캐시에 엔티티가 없으면 데이터베이스에서 엔티티를 조회하고, 1차 캐시에 저장한다. 
3. 조회한 영속 엔티티( `mergeMember` )에 `member` 엔티티의 값을 채워 넣는다.
   - `member` 엔티티의 모든 값 을 `mergeMember` 에 밀어 넣는다. 
     이때 `mergeMember`의 “`회원1`”이라는 이름이 “`회원명변경`”으로 변경된다.
4. 영속 상태인 mergeMember를 반환한다.

참고: 책 자바 ORM 표준 JPA 프로그래밍 3.6.5



**병합시 동작 방식을 간단히 정리**

1. 준영속 엔티티의 식별자 값으로 영속 엔티티를 조회한다. 
2. 영속 엔티티의 값을 준영속 엔티티의 값으로 모두 교체한다.(병합한다.) 
3. 트랜잭션 커밋 시점에 변경 감지 기능이 동작해서 데이터베이스에 UPDATE SQL이 실행



**주의:** 
변경 감지 기능을 사용하면 원하는 속성만 선택해서 변경할 수 있지만, 병합을 사용하면 모든 속성이 변경된다. 
병합시 값이 없으면 null 로 업데이트 할 위험도 있다. (병합은 모든 필드를 교체한다.)



### 4. 상품 리포지토리의 저장 메서드 분석 `ItemRepository`

```java
package jpabook.jpashop.repository;

@Repository
public class ItemRepository {
	@PersistenceContext
 	EntityManager em;
 
    public void save(Item item) {
 		if (item.getId() == null) {
 			em.persist(item);
 		} else {
 			em.merge(item);
 		}
 	}
 //...
}
```

- `save()` 메서드는 식별자 값이 없으면( null ), 새로운 엔티티로 판단해서 영속화(persist)하고 식별자가 있으면 병합(merge)
- 지금처럼 준영속 상태인 상품 엔티티를 수정할 때는 `id` 값이 있으므로 병합 수행



**새로운 엔티티 저장과 준영속 엔티티 병합을 편리하게 한번에 처리**

상품 리포지토리에선 `save()` 메서드를 유심히 봐야 하는데, 이 메서드 하나로 저장과 수정(병합)을 다 처리한다. 코드를 보면 식별자 값이 없으면 새로운 엔티티로 판단해서 `persist()` 로 영속화하고 만약 식별자 값이 있으면 이미 한번 영속화 되었던 엔티티로 판단해서 `merge()` 로 수정(병합)한다. 결국 여기서의 저장 (save)이라는 의미는 신규 데이터를 저장하는 것뿐만 아니라 변경된 데이터의 저장이라는 의미도 포함한다.  이렇게 함으로써 이 메서드를 사용하는 클라이언트는 저장과 수정을 구분하지 않아도 되므로 클라이언트의 로직이 단순해진다.

여기서 사용하는 수정(병합)은 준영속 상태의 엔티티를 수정할 때 사용한다. 영속 상태의 엔티티는 변경 감 지(dirty checking)기능이 동작해서 트랜잭션을 커밋할 때 자동으로 수정되므로 별도의 수정 메서드를 호 출할 필요가 없고 그런 메서드도 없다.



**참고:**
`save()` 메서드는 식별자를 자동 생성해야 정상 동작한다. 여기서 사용한 `Item` 엔티티의 식별자는 자동으로 생성되도록 `@GeneratedValue` 를 선언했다. 따라서 식별자 없이 `save()` 메서드를 호출하면 `persist()` 가 호출되면서 식별자 값이 자동으로 할당된다. 반면에 식별자를 직접 할당하도록 `@Id` 만 선언 했다고 가정하자. 이 경우 식별자를 직접 할당하지 않고, `save()` 메서드를 호출하면 식별자가 없는 상태로 `persist()` 를 호출한다. 그러면 식별자가 없다는 예외가 발생한다.



**참고:** 
실무에서는 보통 업데이트 기능이 매우 제한적이다. 그런데 병합은 모든 필드를 변경해버리고, 데이터 가 없으면 `null` 로 업데이트 해버린다. 병합을 사용하면서 이 문제를 해결하려면, 변경 폼 화면에서 모든 데이터를 항상 유지해야 한다. 실무에서는 보통 변경가능한 데이터만 노출하기 때문에, 병합을 사용하는 것이 오히려 번거롭다.



**가장 좋은 해결 방법:**
엔티티를 변경할 때는 **항상 변경 감지를 사용**하세요.

1. 컨트롤러에서 어설프게 엔티티를 생성하지 마세요.
2. 트랜잭션이 있는 서비스 계층에 식별자( id )와 변경할 데이터를 명확하게 전달하세요.(파라미터 or dto) 
3. 트랜잭션이 있는 서비스 계층에서 영속 상태의 엔티티를 조회하고, 엔티티의 데이터를 직접 변경하세요. 
4. 트랜잭션 커밋 시점에 변경 감지가 실행됩니다.



```java
@Controller
@RequiredArgsConstructor
public class ItemController {
 
    private final ItemService itemService;
 
    /**
    * 상품 수정, 권장 코드
 	*/
 	@PostMapping(value = "/items/{itemId}/edit")
 	public String updateItem(@PathVariable Long itemId, @ModelAttribute("form") BookForm form) {
        itemService.updateItem(itemId, form.getName(), form.getPrice(), form.getStockQuantity());
        return "redirect:/items";
    }
}
```



```java
package jpabook.jpashop.service;

@Service
@RequiredArgsConstructor
public class ItemService {

    private final ItemRepository itemRepository;

    /**
	 * 영속성 컨텍스트가 자동 변경
	 */
	 @Transactional
	 public void updateItem(Long id, String name, int price, int stockQuantity)
	{
		 Item item = itemRepository.findOne(id);
		 item.setName(name);
		 item.setPrice(price);
 		 item.setStockQuantity(stockQuantity);
	 }
}
```

