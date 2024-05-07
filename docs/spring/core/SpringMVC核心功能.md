# 验证框架

| 区别         | @Valid                                          | @Validated              |
| :----------- | :---------------------------------------------- | :---------------------- |
| 提供者       | JSR-303 规范                                    | Spring                  |
| 是否支持分组 | 不支持                                          | 支持                    |
| 标注位置     | METHOD, FIELD, CONSTRUCTOR, PARAMETER, TYPE_USE | TYPE, METHOD, PARAMETER |
| 嵌套校验     | 支持                                            | 不支持                  |

- @Validated
  - 当注解在在方法参数上时，和 @Valid 语义一致，用来处理 @RequestBody 注解的方法参数
  - 当注解在类上时，可以处理 @RequestParam 和 @PathVariable 注解，同 javax.validation.constraints 注解联合使用
  - 支持group分组验证
- @Valid
  - 当注解在属性上时，可以用于校验父对象时的级联校验



# 全局异常处理

- 全局异常处理类注解@RestControllerAdvice

- ConstraintViolationException是javax异常，处理 @RequestParam 和 @PathVariable 注解的请求参数异常

- MethodArgumentNotValidException是Spring异常，处理 @RequestBody 注解的请求参数异常

- 多个请求参数，只要有一个校验失败，后面的其他参数就不做校验；对@RequestParam和@RequestBody都起作用

  ```java
  @Bean
  public Validator validator() {
          ValidatorFactory validatorFactory = Validation.byProvider(HibernateValidator.class)
                  .configure()
                  .failFast(true) // failFast 只要出现校验失败的情况，就立即结束校验，不再进行后续校验
                  .buildValidatorFactory();
          
          return validatorFactory.getValidator();
  }
  ```

  
