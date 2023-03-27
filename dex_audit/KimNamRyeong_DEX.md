## 김남령

## 1. Remove Liquidity Does Not Work on Proper Request

### 설명

문제 코드: removeLiquidity 함수, 151번 줄

```jsx
function removeLiquidity ... {
	require(amountX >_minimumTokenXAmount && amountY>_minimumTokenYAmount, "INSUFFICIENT_LIQUIDITY_BURNED");
}
```

miniminTokenAmount를 본인이 유동성 공급한 토큰만큼으로 설정할 시 에러를 발생시킨다.

### 파급력

Informational

-   산정 이유
    -   컨트랙트의 효용성이 떨어지는 문제만 발생하기 때문에 Informational로 산정했다.

### 해결방안

-   조건문에 등호를 추가한다.

```jsx
function removeLiquidity ... {
	require(amountX >= _minimumTokenXAmount && amountY >= _minimumTokenYAmount, "INSUFFICIENT_LIQUIDITY_BURNED");
}
```
