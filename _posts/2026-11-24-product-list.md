---
layout: post
title: "상품 목록 페이지는 어떻게 동작할까: JavaScript로 필터·정렬·페이지네이션 구현하기"
author: 양설아
type: Explanation
order: 1
---

## 들어가며

이화 대동제에는 여러 부스가 참여하고, 각 부스는 서로 다른 상품을 판매한다. 

방문자는 축제 현장에서 무엇을 판매하는지 확인하고 싶어도 부스마다 직접 찾아가거나 흩어진 정보를 따로 확인해야 하는 불편함을 겪는다.

특히 판매 상품이 많아지면 원하는 음식이나 가까운 판매 장소를 찾는 데 시간이 오래 걸릴 수 있다.

우리 팀은 이러한 불편을 해결하기 위해 대동제 상품과 부스 정보를 한곳에서 확인할 수 있는 웹 서비스인 **이화 대동제: Ewha Market**을 개발했다. 

이 웹 서비스의 사용자는 대동제에 등록된 전체 상품 목록을 살펴보고, 상품의 카테고리와 판매 장소를 기준으로 필터링해 필요한 정보만 확인할 수 있다. 

상품 카드를 선택하면 상세 페이지로 이동하여 상품의 가격, 부스의 위치, 판매자 연락처 등의 정보를 더 자세히 볼 수도 있다.

이 포스트에서는 이러한 목적을 가진 웹 서비스의 기능 중에서도 상품 목록 페이지가 어떻게 동작하는지에 대해 다룬다.

## 상품 목록 페이지의 구현 목표, 그리고 목표를 실현하기 위한 기능

상품 목록 페이지의 주된 목표는 사용자가 자신의 조건과 우선순위에 맞는 상품을 빠르게 찾을 수 있도록 목록의 탐색 방식을 제공하는 것이었다.

대동제에서 판매하는 상품을 정리한 목록을 웹 페이지로 만들더라도, 등록된 상품의 수가 많아 사용자가 관심 없는 상품들까지 모두 살펴봐야 한다면 상품 목록 페이지가 있어도 탐색 시간은 크게 줄어들지 않는다.

그렇기에 단순히 상품 목록을 보여주는 페이지를 구현하는 것만으로는 타겟 사용자가 겪는 불편 사항을 효과적으로 해결할 수 없다.

그래서 우리 팀은 이 문제를 해결하기 위해 상품 목록에 필터, 정렬, 페이지네이션을 적용하기로 했다.

세 기능은 각각 아래와 같은 역할을 담당한다.

- **필터**는 사용자가 선택한 조건에 맞지 않는 상품을 제외하여 탐색 범위를 줄인다.
- **정렬**은 사용자가 중요하게 생각하는 기준에 따라 상품의 순서를 바꾼다.
- **페이지네이션**은 한 화면에 표시되는 상품 수를 제한하여 정보량과 탐색 부담을 줄인다.

상품 목록 페이지에서는 상품 카테고리와 판매 장소를 함께 필터링할 수 있다. 정렬은 최신순, 가격 낮은순, 가격 높은순을 제공한다. 페이지당 최대 8개의 상품 카드를 표시하며, 나머지 상품은 다음 페이지에서 확인할 수 있다.

페이지를 다시 불러오지 않고도 목록이 즉시 바뀌도록 주요 동작은 JavaScript로 구현했다.

## 구현 단계

### 1. 상품 정보를 `data-*` 속성으로 저장하기

필터와 정렬을 실행하려면 JavaScript가 각 상품의 카테고리, 장소, 가격을 읽을 수 있어야 한다. 서버에서 전달받은 상품 정보를 카드의 `data-*` 속성에 저장했다.

{% raw %}
```html
<article
  class="product-list-card"
  data-name="{{ item_name }}"
  data-category="{{ item_category }}"
  data-address="{{ item_address }}"
  data-price="{{ item.price|default(0, true) }}"
>
  ...
</article>
```
{% endraw %}

화면에는 상품 이름과 태그를 표시하면서, JavaScript는 `dataset.category`, `dataset.address`, `dataset.price`를 통해 같은 정보를 사용할 수 있다. 별도의 데이터 배열을 다시 만들지 않아도 렌더링된 카드 자체가 필터와 정렬의 대상이 된다.

### 2. 사용자의 선택을 하나의 상태로 관리하기

카테고리, 장소, 정렬 기준은 서로 독립적으로 바뀔 수 있다. 현재 선택값을 `state` 객체에 저장하여 어떤 기능이 실행되더라도 동일한 상태를 기준으로 목록을 다시 계산하도록 했다.

```javascript
const state = {
  category: null,
  address: null,
  sort: "최신순"
};
```

카테고리나 장소가 `null`이면 해당 조건을 적용하지 않는다. 예를 들어 카테고리만 `분식`으로 선택했다면 장소와 관계없이 모든 분식 상품을 보여준다. 이후 장소로 `학생문화관`까지 선택하면 두 조건을 모두 만족하는 상품만 남는다.

### 3. 카테고리와 장소를 함께 필터링하기

상품을 필터링할 때는 `Array.prototype.filter()`를 사용했다.

```javascript
function filteredCards() {
  const filtered = cards.filter(card => {
    const matchesCategory =
      !state.category ||
      card.dataset.category === state.category;

    const matchesAddress =
      !state.address ||
      card.dataset.address === state.address;

    return matchesCategory && matchesAddress;
  });

  return filtered.sort((a, b) => {
    if (state.sort === "가격 낮은순") {
      return Number(a.dataset.price) - Number(b.dataset.price);
    }

    if (state.sort === "가격 높은순") {
      return Number(b.dataset.price) - Number(a.dataset.price);
    }

    return Number(a.dataset.originalIndex)
      - Number(b.dataset.originalIndex);
  });
}
```

`matchesCategory`와 `matchesAddress`를 각각 계산한 뒤 두 값이 모두 참인 카드만 반환한다. 이 구조를 사용하면 카테고리와 장소 중 하나만 선택하거나 두 조건을 함께 선택하는 경우를 같은 함수에서 처리할 수 있다.

필터링된 배열에 바로 정렬을 적용했다. 가격은 HTML의 `data-*` 속성에서 문자열로 읽히기 때문에 `Number()`로 변환한 뒤 비교한다.

최신순은 현재 서버에서 전달받아 처음 표시한 순서를 기준으로 한다. 이를 위해 페이지가 처음 열릴 때 각 카드에 원래 인덱스를 저장했다.

```javascript
cards.forEach((card, index) => {
  card.dataset.originalIndex = index;
});
```

### 4. 선택한 정렬 기준 반영하기

사용자가 정렬 메뉴에서 다른 항목을 선택하면 상태를 갱신하고 첫 페이지부터 다시 표시한다.

```javascript
sortSelect.addEventListener("change", () => {
  state.sort = sortSelect.value;
  history.replaceState({}, "", pageUrl(1));
  showPage(1);
});
```

가격 낮은순과 높은순은 같은 데이터를 사용하지만 비교식의 순서를 반대로 적용한다. 정렬할 때 서버에 다시 요청하지 않으므로 선택 결과가 즉시 화면에 나타난다.

필터가 적용된 상태에서도 정렬이 유지된다. 먼저 조건에 맞는 상품만 남긴 뒤 그 결과를 정렬하기 때문이다.

### 5. 페이지당 8개의 상품만 보여주기

한 화면에 너무 많은 상품이 표시되지 않도록 페이지당 상품 수를 8개로 정했다.

```javascript
const ITEMS_PER_PAGE = 8;

function showPage(page) {
  const visibleCards = filteredCards();
  const totalPages =
    Math.ceil(visibleCards.length / ITEMS_PER_PAGE);

  const safePage =
    totalPages === 0 ? 1 : Math.min(page, totalPages);

  const start = (safePage - 1) * ITEMS_PER_PAGE;
  const end = start + ITEMS_PER_PAGE;

  cards.forEach(card => {
    card.hidden = true;
  });

  visibleCards.forEach((card, index) => {
    grid.append(card);
    card.hidden = index < start || index >= end;
  });

  renderPagination(safePage, totalPages);
}
```

현재 페이지가 1이라면 배열의 0번째부터 7번째 카드까지 보여준다. 2페이지에서는 8번째 카드부터 보여준다. 전체 페이지 수는 필터링된 상품 수를 기준으로 다시 계산한다.

따라서 필터를 적용해 결과가 8개 이하가 되면 페이지 번호도 1개로 줄어든다. 필터 결과가 없으면 빈 결과 안내를 표시한다.

### 6. 필터 변경 시 첫 페이지로 이동하기

사용자가 2페이지를 보고 있는 상태에서 필터를 변경하면 결과 상품 수가 줄어들 수 있다. 이때 기존 2페이지를 유지하면 상품이 없는 화면이 나타날 수 있다.

이를 방지하기 위해 필터가 변경될 때 현재 페이지를 1로 초기화했다.

```javascript
state[group.dataset.filterType] =
  wasPressed ? null : button.textContent.trim();

history.replaceState({}, "", pageUrl(1));
showPage(1);
```

선택한 필터 버튼에는 `aria-pressed`를 적용했다. 화면의 색상뿐 아니라 접근성 정보에서도 현재 선택 상태를 구분할 수 있다.

### 7. URL과 브라우저 이동 기록 유지하기

페이지 번호를 누를 때마다 전체 문서를 다시 불러오는 대신 History API를 사용해 URL과 목록만 변경했다.

```javascript
history.pushState({}, "", link.href);
showPage(page, true);
```

현재 페이지는 URL의 `page` 값을 읽어 결정한다.

```javax
function getRequestedPage(totalPages) {
  const page = Number(
    new URLSearchParams(window.location.search).get("page")
  );

  return Number.isInteger(page) &&
    page >= 1 &&
    page <= totalPages
      ? page
      : 1;
}
```

브라우저의 이전 또는 다음 버튼을 눌렀을 때는 `popstate` 이벤트가 발생한다. 이때 URL의 페이지 번호를 다시 읽어 목록을 갱신한다.

```javascript
window.addEventListener("popstate", () => {
  const totalPages =
    Math.ceil(filteredCards().length / ITEMS_PER_PAGE);

  showPage(getRequestedPage(totalPages));
});
```

이 방식을 사용하면 페이지를 새로 불러오지 않아도 주소창의 페이지 번호와 실제 화면이 일치한다.

## 결과

페이지네이션이 적용된 상품 목록의 한 페이지에는 최대 8개의 상품이 표시된다. 필터 결과가 달라지면 상품 수와 페이지 수도 함께 다시 계산된다.

![페이지네이션이 적용된 상품 목록: 페이지 1, 2]({{"/assets/images/product-list-pagination-applied.png" | relative_url}})

상품 목록 페이지에서 사용자는 상품 카테고리와 판매 장소를 선택할 수 있다. 두 조건을 함께 적용하면 해당 조건을 모두 만족하는 상품만 남는다. 정렬 기준을 변경하면 현재 필터 결과 안에서 상품 순서가 즉시 바뀐다.

![장소와 상품 카테고리 필터를 적용한 화면]({{"assets/images/product-list-filter-applied.png" | relative_url}})

최신순, 가격이 낮은 순, 가격이 높은 순 세 가지의 정렬 조건으로 상품을 정렬하는 것도 가능하다.

![상품 목록이 최신순, 가격 낮은 순, 가격 높은 순으로 정렬된 화면]({{"assets/images/product-list-sorting-applied.png" | relative_url}})

상품 카드의 이미지, 태그, 상품명, 판매자, 운영시간 영역을 누르면 해당 상품의 상세 페이지로 이동한다. 카드 안의 좋아요 버튼은 상세 이동과 분리되어 독립적으로 작동한다.

![상품 하나를 클릭해 상세 정보가 보이는 화면]({{"assets/images/product-list-detail.png" | relative_url}})


## 배운 점

이번 구현을 통해 목록 페이지의 사용성은 상품을 보기 좋게 배치하는 것만으로 완성되지 않는다는 점을 배웠다. 

사용자가 원하는 정보에 도달하는 데 들이는 수고를 줄이기 위해서는 구체적인 기준을 정해 필터, 정렬, 페이지네이션이라는 기능을 활용해야 한다.

특히 각 기능이 따로 실행되는 것처럼 보여도 실제로는 하나의 상태를 공유해야 했다는 점을 배울 수 있었다. 예시로 필터가 바뀌면 정렬과 페이지네이션도 같은 결과를 기준으로 다시 계산해야 한다. 이를 위해 사용자 선택을 `state` 객체에 저장하고 `filteredCards()`와 `showPage()`를 중심으로 화면을 갱신했다.

URL과 화면 상태를 일치시키는 것도 중요했다. `pushState()`, `replaceState()`, `popstate`를 사용하면서 새로고침 없는 화면 전환과 브라우저 탐색 기록을 함께 관리하는 방법을 이해할 수 있었다.

현재 최신순은 서버에서 전달받은 상품의 초기 순서를 기준으로 한다. 앞으로 실제 등록 시간을 기준으로 정확하게 정렬하려면 데이터베이스에 등록 시각을 저장하고 그 값을 비교해야 한다. 이처럼 화면에서 제공하는 정렬 기준은 데이터 구조와 함께 설계해야 한다는 점도 알게 되었다.

## 참고자료

- [팀 어서오소 프로젝트 저장소](https://github.com/devleeji/2026_osp_project_welcome)
- [MDN: Array.prototype.filter()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter)
- [MDN: Array.prototype.sort()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort)
- [MDN: URLSearchParams](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams)
- [MDN: History API](https://developer.mozilla.org/en-US/docs/Web/API/History)
- [MDN: popstate event](https://developer.mozilla.org/en-US/docs/Web/API/Window/popstate_event)
