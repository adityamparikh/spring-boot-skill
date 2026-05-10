# Exception Handling and ProblemDetail

Use a single `@RestControllerAdvice` to translate domain exceptions into RFC 7807 `ProblemDetail` responses. Do not return ad-hoc error JSON.

```java
@RestControllerAdvice
class ApiExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    ProblemDetail onNotFound(OrderNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setType(URI.create("https://example.com/problems/order-not-found"));
        pd.setProperty("orderId", ex.getOrderId());
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail onValidation(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
        pd.setProperty("fieldErrors",
            ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
                .toList());
        return pd;
    }
}
```

## Rules

- Map known domain exceptions to specific status codes.
- Let unmapped exceptions fall through to Spring Boot's default `ProblemDetail` handler.
- Never catch `Exception.class` and swallow it -- only catch broadly when re-throwing or adding real recovery.
