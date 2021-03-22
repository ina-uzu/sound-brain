## Composite Types



### Slice

* 구성요소 : pointer, len, capacity 

* 배열과 달리 == 연산자로 원소 값 비교 불가

  `slice1 == slice2` 는 각 슬라이스가 동일한 포인터(주소값)를 가지는지를 비교하는 것으로, 값 비교가 아니다

* 슬라이스와 배열에서 == 연산자의 일관성 없는 취급은 혼동을 유발할 수 있으므로, 슬라이스에는 nil과의 비교만 유일하게 허용

**Zero vs Nil Slice** 

```
var s []int			// len(s) ==0 , s == nil
s = nil					// len(s) ==0 , s == nil
s = []int(nil)	// len(s) ==0 , s == nil
s = []int{}			// len(s) ==0 , s != nil
```

---

### Map

* Key는 == 연산자로 비교 가능 한 타입만 가능하다 
* 안되는 타입의 경우 string으로 바꿔서 키로 적용하는 편법이 있다

**nil map** 

* 넣고, 빼는 연산을 하면 바로 패닉 (ok로 거를 수도 없음)

**not nil map**

* val, ok = map["key"]

  값 없으면 해당 타입의 Zero value가 assign되고  패닉은 발생하지 않음


---

### struct

* 필드 안에 비교가 안되는 타입이 하나라도 있으면 == 연산자로 비교 불가
