

**Assert**와 **Guard**는 런타임에 "이 조건은 반드시 참이어야 한다"를 강제하는 방어 코드입니다.

### 1. Assert (단언문)

조건이 거짓이면 **즉시 에러를 던져서 실행을 멈춤**

```typescript
// 간단한 assert 함수
function assert(condition: boolean, message: string): asserts condition {
  if (!condition) {
    throw new Error(`Assertion failed: ${message}`);
  }
}

// 사용 예시
function processPayment(amount: number) {
  assert(amount >= 0, '결제 금액은 0 이상이어야 함');
  assert(Number.isFinite(amount), '결제 금액은 유한한 숫자여야 함');
  
  // 이 지점에 도달했다면 amount는 유효함이 보장됨
  return chargeCard(amount);
}
```

### 2. Guard (가드)

조건을 검사해서 **조기에 리턴하거나 타입을 좁힘**

```typescript
// Early return guard
function calculateDiscount(orderTotal: number, discountRate: number) {
  // 가드 조건: 잘못된 입력은 안전한 기본값 반환
  if (orderTotal < 0 || discountRate < 0 || discountRate > 1) {
    console.error('Invalid discount calculation input');
    return 0; // 안전한 기본값
  }
  
  return orderTotal * discountRate;
}

// Type Guard (타입 좁히기)
function isValidOrder(order: unknown): order is Order {
  return (
    typeof order === 'object' &&
    order !== null &&
    'orderId' in order &&
    'total' in order &&
    typeof order.total === 'number' &&
    order.total >= 0
  );
}

function processOrder(data: unknown) {
  if (!isValidOrder(data)) {
    throw new Error('Invalid order data');
  }
  
  // 이 시점부터 data는 Order 타입으로 좁혀짐
  console.log(data.orderId, data.total);
}
```

---

## 실전 예시: 결제 플로우

```typescript
// 1. 불변식(Invariant) 정의
class PaymentInvariantError extends Error {
  constructor(message: string) {
    super(`결제 불변식 위반: ${message}`);
    this.name = 'PaymentInvariantError';
  }
}

// 2. Assert 헬퍼
function assertPaymentValid(
  total: number,
  discount: number,
  finalAmount: number
): void {
  // 불변식 1: 금액은 모두 0 이상
  assert(total >= 0, '상품 합계는 0 이상이어야 함');
  assert(discount >= 0, '할인액은 0 이상이어야 함');
  assert(finalAmount >= 0, '최종 결제 금액은 0 이상이어야 함');
  
  // 불변식 2: 할인액은 상품 합계를 초과할 수 없음
  if (discount > total) {
    throw new PaymentInvariantError(
      `할인액(${discount})이 상품합계(${total})를 초과함`
    );
  }
  
  // 불변식 3: 최종 금액 = 상품합계 - 할인액
  const expected = total - discount;
  if (Math.abs(finalAmount - expected) > 0.01) { // 부동소수점 오차 허용
    throw new PaymentInvariantError(
      `금액 불일치: 예상=${expected}, 실제=${finalAmount}`
    );
  }
}

// 3. Guard로 경계 보호
interface PaymentRequest {
  items: Array<{ price: number; quantity: number }>;
  discountCode?: string;
}

function processPayment(request: PaymentRequest) {
  // Guard 1: 입력 검증
  if (!request.items || request.items.length === 0) {
    return { success: false, error: '상품이 없습니다' };
  }
  
  // Guard 2: 각 아이템 검증
  for (const item of request.items) {
    if (item.price < 0 || item.quantity <= 0) {
      return { 
        success: false, 
        error: '잘못된 상품 정보' 
      };
    }
  }
  
  // 계산
  const total = request.items.reduce(
    (sum, item) => sum + item.price * item.quantity, 
    0
  );
  const discount = calculateDiscount(request.discountCode);
  const finalAmount = total - discount;
  
  // Assert: 불변식 검증
  try {
    assertPaymentValid(total, discount, finalAmount);
  } catch (error) {
    // 불변식 위반은 심각한 버그 → 로깅 + 안전한 실패
    console.error('Payment invariant violation', error);
    return { 
      success: false, 
      error: '결제 처리 중 오류가 발생했습니다' 
    };
  }
  
  // 실제 결제 진행
  return chargeCard({ amount: finalAmount });
}
```

---

## 실무 패턴: 서버 응답 검증

```typescript
import { z } from 'zod';

// Zod 스키마 (컴파일 타임 + 런타임)
const OrderSchema = z.object({
  orderId: z.string().uuid(),
  status: z.enum(['pending', 'paid', 'shipped', 'delivered']),
  total: z.number().min(0),
  discount: z.number().min(0),
});

type Order = z.infer<typeof OrderSchema>;

// 상태 전이 가드 (비즈니스 로직)
const VALID_STATUS_TRANSITIONS: Record<Order['status'], Order['status'][]> = {
  pending: ['paid'],
  paid: ['shipped'],
  shipped: ['delivered'],
  delivered: [], // 최종 상태
};

function assertValidStatusTransition(
  from: Order['status'],
  to: Order['status']
): void {
  const allowed = VALID_STATUS_TRANSITIONS[from];
  
  if (!allowed.includes(to)) {
    throw new Error(
      `잘못된 주문 상태 전이: ${from} → ${to} (허용: ${allowed.join(', ')})`
    );
  }
}

// API 응답 처리
async function updateOrderStatus(orderId: string, newStatus: Order['status']) {
  // 1. 현재 주문 조회
  const response = await fetch(`/api/orders/${orderId}`);
  const rawData = await response.json();
  
  // 2. Guard: 스키마 검증 (Zod)
  const parseResult = OrderSchema.safeParse(rawData);
  if (!parseResult.success) {
    console.error('Invalid order data', parseResult.error);
    throw new Error('서버 응답이 올바르지 않습니다');
  }
  
  const order = parseResult.data;
  
  // 3. Assert: 상태 전이 불변식 검증
  assertValidStatusTransition(order.status, newStatus);
  
  // 4. 업데이트 진행
  return updateOrder(orderId, { status: newStatus });
}
```

---

## 핵심 차이점

||**Assert**|**Guard**|
|---|---|---|
|**목적**|"이 조건은 **반드시** 참이다"|"이 조건이 아니면 **안전하게 처리**"|
|**실패 시**|에러 던짐 (프로그램 멈춤)|조기 리턴 / 기본값 / 타입 좁히기|
|**사용 위치**|내부 로직 (불변식 검증)|경계 (입력 검증)|
|**예시**|`assert(discount <= total)`|`if (!isValid) return null;`|

---

## 실무 팁

```typescript
// ✅ 좋은 예: 경계에서 Guard, 내부에서 Assert
function checkout(cartData: unknown) {
  // Guard: 외부 입력 검증
  if (!isValidCart(cartData)) {
    return { error: 'Invalid cart' };
  }
  
  const total = calculateTotal(cartData);
  const discount = getDiscount();
  
  // Assert: 내부 불변식 검증
  assert(discount <= total, 'Discount exceeds total');
  
  return processPayment(total - discount);
}

// ❌ 나쁜 예: 조용히 무시하거나 잘못된 상태 진행
function checkoutBad(cartData: any) {
  const total = calculateTotal(cartData); // cartData가 invalid여도 진행
  const discount = getDiscount();
  
  // 할인이 합계보다 크면 조용히 0으로 만듦 (버그 숨김)
  const finalAmount = Math.max(0, total - discount);
  
  return processPayment(finalAmount);
}
```

궁금한 점 있으면 말씀해주세요! 특정 시나리오(딥링크 검증, 스토리지 데이터 등)에 대한 예시가 필요하면 추가로 보여드릴게요.