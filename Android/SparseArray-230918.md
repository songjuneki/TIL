# SparseArray

- [SparseArray](#sparsearray)
  - [개요](#개요)
  - [SparseArray 란](#sparsearray-란)
  - [그렇다면 HashMap은?](#그렇다면-hashmap은)
  - [결론](#결론)

---

## 개요

Google이 Jetpack Navigation 2.4.0 이전 버전을 위해서 확장 함수로 구현한 Multiple Navigation 이 있다. [코드 보기](https://github.com/android/architecture-components-samples/blob/8f4936b34ec84f7f058fba9732b8692e97c65d8f/NavigationAdvancedSample/app/src/main/java/com/example/android/navigationadvancedsample/NavigationExtensions.kt)  
해당 코드를 분석해보다가 SparseArray 란게 눈에 띄었다. 사용하는 코드를 보니 Map 과 같은 형태의 자료구조인 것 같은데 뭐가 다른지 좀 더 알아보기로 했다.

## SparseArray 란

안드로이드 공식 문서에서 설명하는 [SparseArray](https://developer.android.com/reference/android/util/SparseArray)이다.

> SparseArray maps integers to Objects and, unlike a normal array of Objects, its indices can contain gaps. SparseArray is intended to be more memory-efficient than a HashMap, because it avoids auto-boxing keys and its data structure doesn't rely on an extra entry object for each mapping.
>
> Note that this container keeps its mappings in an array data structure, using a binary search to find keys. The implementation is not intended to be appropriate for data structures that may contain large numbers of items. It is generally slower than a HashMap because lookups require a binary search, and adds and removes require inserting and deleting entries in the array. For containers holding up to hundreds of items, the performance difference is less than 50%.
>
> To help with performance, the container includes an optimization when removing keys: instead of compacting its array immediately, it leaves the removed entry marked as deleted. The entry can then be re-used for the same key or compacted later in a single garbage collection of all removed entries. This garbage collection must be performed whenever the array needs to be grown, or when the map size or entry values are retrieved.
>
> ...

SparseArray 는 Object를 정수형으로 키로 매핑한 자료 구조이다. 일반적인 Array와 달리 공백이 포함될 수 있다고 한다. 그래서 Sparse 라는 이름을 붙인 것이다.  
또 *auto-boxing 을 피하기 때문에 HashMap 보다 메모리 효율이 좋다고 한다.

> auto-boxing : 자바의 Collection 에 원시 타입(int, double, boolean 등)을 저장하기 위해서 Wrapper 클래스(Integer, Double, Boolean 등)으로 변환하여 저장해야하는데 이를 컴파일러가 자동으로 해주는 것

하지만 내부에서 조회에 이진 검색을 사용하고 추가/제거 시에는 배열의 항목을 삽입 및 삭제해야 하므로 일반적으로 HashMap 보다 느리다고 한다. 그래서 많은 양의 데이터를 저장하기에는 적합하지 않다고 한다.

성능을 위해서 키를 제거할 때 배열에서 바로 제거하는 것이 아닌 deleted 상태로 표시하여 최적화 했다고 한다. 이후에 내부에서 garbage collection 을 사용하여 배열에서 아예 없앤다고 한다. 이 garbage collection 은 배열을 늘릴 때나 size를 측정할 때나 값을 조회할 때마다 수행한다고 한다.

---

더 자세히 알아보기 위해서 AOSP에 정의된 코드를 찾아보았다. [링크](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/util/SparseArray.java)

```java
public class SparseArray<E> implements Cloneable {
    private static final Object DELETED = new Object();
    private boolean mGarbage = false;
    private int[] mKeys;
    private Object[] mValues;
    private int mSize;

    ...

    public void delete(int key) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            if (mValues[i] != DELETED) {
                mValues[i] = DELETED;
                mGarbage = true;
            }
        }
    }

    public void remove(int key) {
        delete(key);
    }

    public void set(int key, E value) {
        put(key, value);
    }

    public void put(int key, E value) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            mValues[i] = value;
        } else {
            i = ~i;

            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }

            if (mGarbage && mSize >= mKeys.length) {
                gc();

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }

            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++;
        }
    }

    ...
}

```

먼저 삽입/삭제 하는 코드이다. 공식 문서에서 설명한대로 내부에서 조회할 때마다 이진 탐색을 사용하여 인덱스 값을 구하고 삽입 및 삭제를 하는 구조이다.

먼저 삭제할 때 값 배열에서 해당 인덱스를 미리 정의된 DELETED 로 표시(교체)한다. 그리고 mGarbage를 true로 바꿔준다. 이 mGarbage는 garbage collection 을 활성화 하기 위한 플래그로 사용된다.

삽입할 때를 보자 인덱스가 있다면 값을 바꿔주고 없다면 이진 탐색 결과 값은 찾을 키가 존재해야 할 인덱스의 값의 ~(비트 반전 연산)을 한 값이 나온다. 이로 인해 음수가 나오고 이를 다시 비트 반전 연산을 해 양수로 바꾼다.  
해당 값이 배열의 크기보다 작다면 값이 DELETED 상태인지 확인을 한다. DELETED 상태라면 값을 위와 동일하게 바꿔주기만 하면 된다.  
DELETED 상태가 아니라면 배열에 새로 삽입하는 것인데 `GrowingArrayUtils.insert()` 함수를 통해 해당 인덱스 값에 키와 값을 각각 삽입한다. 이 함수는 내부에서 배열을 복제하고 삽입하기 때문에 HashMap 보다 느리다고 한 것이다.  
이 삽입 전에 mGarbage 가 활성화 돼있다면 garbage collection 을 실행하고 다시 인덱스 값을 구하고 삽입한다.

또 위에서 말한 garbage collection은 `put()` 을 이용한 삽입 말고도 `append()` 를 통한 삽입이나 `size()` 크기를 측정하는 함수, `valueAt()`, `keyAt()`, ... 과 같은 탐색 함수를 통해서도 이루어진다.

이 garbage collection 의 내부 구현은 다음과 같다.

```java
    private void gc() {
        int n = mSize;
        int o = 0;
        int[] keys = mKeys;
        Object[] values = mValues;

        for (int i = 0; i < n; i++) {
            Object val = values[i];

            if (val != DELETED) {
                if (i != o) {
                    keys[o] = keys[i];
                    values[o] = val;
                    values[i] = null;
                }

                o++;
            }
        }

        mGarbage = false;
        mSize = o;
    }
```

SparseArray의 크기를 뜻하는 mSize를 다시 측정하는데 배열을 처음부터 끝까지 순회하여 값이 DELETED 가 아니면 크기에 포함시키는 로직이다. 그 과정에서 키와 값 배열의 인덱스를 일치하게 정리하고 값을 DELETED 대신 null 로 바꾼다.

## 그렇다면 HashMap은?

HashMap은 해싱(Hashing)을 이용하여 값과 1:1로 매핑을 한다. 이름에 Hash가 들어가는 이유이다. 이렇기 때문에 키를 해싱하는 해시 함수와 매핑된 값이 들어있는 공간인 Bucket, 또 해시 충돌을 방지하기 위해 로드팩터를 넘어가면 크기가 2배 커지며 낭비되는 공간과 같은 이유로 HashMap은 메모리를 많이 쓰인다는 것이다.  
하지만 HashMap은 매핑으로 인해 삽입/삭제 및 조회의 시간 복잡도는 평균 O(1)이라는 장점이 있다.

## 결론

수백개의 데이터까지는 SparseArray를 사용하자, 그 이상은 HashMap을 사용하자

SparseArray는 처음에 Map과 같은 자료구조인줄 알았는데 자세히 알아 볼 수록 왜 Array라고 작명했는지 알 수 있었다. 또 모바일이라는 환경에서 리소스를 아껴야 한다는 의도를 잘 볼 수 있었다. 안드로이드에서는 이러한 SparseArray 말고도 유용한 컬렉션, 유틸을 제공한다. ArrayMap, LruCache, CircularArray 등 이러한 것들도 많이 알아보고 앱에 적용을 한다면 만족할 만한 경험을 주는 앱을 만들 수 있을 것이다.
