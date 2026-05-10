# Validation

Bean Validation 3 (Jakarta) is auto-configured by `spring-boot-starter-validation`. Apply constraints at the boundary: request DTOs and `@PathVariable` / `@RequestParam` arguments.

```java
record CreateUserRequest(
    @NotBlank @Size(max = 80) String name,
    @Email String email,
    @Past LocalDate birthDate
) {}

@RestController
@Validated  // enables constraint validation on method-level params (path/query)
class UserController {
    @GetMapping("/users/{id}")
    User get(@PathVariable @NotNull UUID id) { ... }
}
```

## Custom Constraints

Implement `ConstraintValidator<MyAnnotation, T>` and register the annotation with `@Constraint(validatedBy = MyValidator.class)`. Keep validators stateless.

For cross-field rules (e.g., "endDate after startDate"), prefer a class-level constraint over scattering checks across services.
