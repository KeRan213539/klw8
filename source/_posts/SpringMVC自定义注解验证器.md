---
title: SpringMVC自定义注解验证器
date: 2018-04-21 23:35:41
tags: SpringMVC 自定义注解验证器
---
<Excerpt in index | 首页摘要> 
# 背景
大家可能会问,spring MVC支持验证注解,如常用的hibernate-validator,为什么要自己实现一套呢?
最近做一个APP的服务端接口,项目中有自己的业务返回码.spring MVC支持的注解验证器无法设置验证不通过的时候的返回码,各种不方便,所以思前想后还是自己实现了一套.废话不多说,开始正文.
+<!-- more -->
<The rest of contents | 余下全文>
# 状态码枚举
状态码枚举中有两个属性: 状态码 和 对应的默认消息
```
public enum ResponseCodeEnum {

    _001("001", "用户未登录");

    /**
     * @Fields code : 状态码
     */
    private String code;
    
    /**
     * @Fields defaultMessage : 默认消息
     */
    private String defaultMessage;
    
    private ResponseCodeEnum (String code, String defaultMessage) {
	this.code = code;
	this.defaultMessage = defaultMessage;
    }

    @JsonValue  // com.fasterxml.jackson.annotation.JsonValue, 项目中用了 jackson 做为 
                          // springMVC的JSON转换器,该注解表式这个方法的返回值生成到JSON中,其他忽略
    public String getCode() {
        return code;
    }

    public String getDefaultMessage() {
        return defaultMessage;
    }
}
```
# 自定义业务异常
业务数据(客户端提交的)验证不过等各种业务处理中的不通过,统一使用该异常,该异常被捕获后会生成统一格式的消息返回到客户端
```
public class CustomValidatorException extends RuntimeException {

    private static final long serialVersionUID = 5968495544349929856L;
    
    private ResponseCodeEnum statusCode;
    
    private String errorMsg;
    
    public CustomValidatorException (ResponseCodeEnum statusCode, String errorMsg) {
	this.statusCode = statusCode;
	this.errorMsg = errorMsg;
    }

    public ResponseCodeEnum getStatusCode() {
        return statusCode;
    }

    public String getErrorMsg() {
        return errorMsg;
    }

}
```
# 验证器接口
该接口作为验证器注解必须实现的接口,负责真正的验证
```
public interface IAnnotationsValidator {

    public void doValidator(Object object, Annotation annotation) throws CustomValidatorException ;
    
}
```
# 验证器注解
验证器相关注解定义,首先得有几个基础注解
## 基础注解
##### EnableValidator  负责开启验证,使用了该注解的参数Bean才会被验证
```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
public @interface EnableValidator {

}
```
##### CustomValidator 该注解作用于注解(该注解只能被其他注解使用,不能被非注解的类使用),使用了该注解的注解才被做为验证器注解
```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.ANNOTATION_TYPE})
@Documented
public @interface CustomValidator {

}
```
## 验证器注解
这里先只写2个注解吧,其他的可以由其他开发人员开发
##### Required 必传参数注解,只有这个注解验证参数是否有值,其他注解有值才验证,没值直接通过
```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD})
@Documented
@Inherited  // 子类可以继承父类的注解
@CustomValidator
public @interface Required {

    /**
     * @Title: responseCode
     * @Description: 验证失败(不通过)的code
     * @return
     */
    ResponseCodeEnum responseCode();
    
    /**
     * @Title: validatFailMessage
     * @Description: 验证失败(不通过)的文字消息,可为空,默认使用ResponseCodeEnum对应的消息
     * @return
     */
    String validatFailMessage() default "";
    
    /**
     * @Fields validatorSpringBeanName : 此注解对应的验证器的springBean名称,该名称在定义注解的时候写死
     */
    final String validatorSpringBeanName = "requiredValidator";
    
}
```
Required  注解中的几个属性是所有验证器注解都必须有的,大家可能注意到了validatorSpringBeanName , 没错,切面就是根据这个在spring容器中拿验证器实现的
##### NotEmpty 用于验证字符串,List,集合,数组,Map等不能为空,这里的空不包括null,是null以外的空
```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD})
@Documented
@Inherited  // 子类可以继承父类的注解
@CustomValidator
public @interface NotEmpty {

    /**
     * @Title: responseCode
     * @Description: 验证失败(不通过)的code
     * @return
     */
    ResponseCodeEnum responseCode();
    
    /**
     * @Title: validatFailMessage
     * @Description: 验证失败(不通过)的文字消息,可为空,默认使用ResponseStatusCodeEnum对应的消息
     * @return
     */
    String validatFailMessage() default "";
    
    /**
     * @Fields validatorSpringBeanName : 此注解对应的验证器的springBean名称,该名称在定义注解的时候写死
     */
    final String validatorSpringBeanName = "notEmptyValidator";
    
}
```
除了上面两个验证器中的3个必须有的属性,还可以定义其他的属性,比如验证字符串长度的验证器,可以加一个长度的属性. 这些属性可以在验证器实现中获取
## 验证器(注解)实现
##### RequiredImpl: Required 的实现
```
@Component("requiredValidator")
public class RequiredImpl implements IAnnotationsValidator {

    @Override
    public void doValidator(Object object, Annotation annotation) throws CustomValidatorException {
        Required notEmpty = (Required) annotation;
        ResponseStatusCodeEnum statusCode = notEmpty.responseStatusCode();
        String message = notEmpty.validatFailMessage();
        // TODO 获取验证器注解中的其他属性
        // TODO 验证,如果验证不通过,抛出 CustomValidatorException 
    }

}
```
##### NotEmptyImpl: NotEmpty 的实现,具体参考 Required 的实现
# AOP
自定义验证器的核心实现,没有它,上面的东西全是白费
```
public class ValidatorAdvise {
    
    private static Logger logger = LoggerFactory.getLogger(ValidatorAdvise .class);

    public Object validator(ProceedingJoinPoint pjp) {
	// 获取被拦截的方法的参数
	Object[] args = pjp.getArgs();
	// 遍历该方法的所有参数
	if (args != null && args.length > 0) {
	    for (Object arg : args) {
		Class<?> argClassz = arg.getClass();
		if (argClassz.getAnnotation(EnableValidator.class) != null) { // 只有当该参数有EnableValidator注解,也就是开启了验证才处理
		    List<Field> fieldList = getAllFields(null, argClassz); // 获取所有字段
		    // 遍历所有字段,并找出有注解的
		    for (Field field : fieldList) {
			// 检查每个字段的注解,有注解的才处理
			Annotation[] fieldAnns = field.getAnnotations();
			if (fieldAnns != null && fieldAnns.length > 0) {
			    // 遍历该字段的注解,找到验证器的注解
			    for (Annotation fieldAnn : fieldAnns) {
				try {
				    // 检查该注解是否有@CustomValidator,有就说明是验证器
				    if (fieldAnn.annotationType().getAnnotation(CustomValidator.class) != null) {
					// 通过反射拿验证器的springBeanName字段,不为null才处理
					Field validatorSpringBeanNameFiled = fieldAnn.annotationType().getDeclaredField("validatorSpringBeanName");
					if (validatorSpringBeanNameFiled != null) {
					    // 通过spring拿到验证器进行验证,先拿验证器的springBeanName
					    Object validatorSpringBeanName = validatorSpringBeanNameFiled.get(fieldAnn);
					    if(StringUtil.isNotNull(validatorSpringBeanName)) {
						// 名字有值,从spring容器中拿对应的验证器
						IAnnotationsValidator annotationsValidator = SystemApplicationContext.SPRING_CONTEXT.getBean((String)validatorSpringBeanName, IAnnotationsValidator.class);
						if(annotationsValidator != null) {
						    // 验证器不为空,调用验证器
						    field.setAccessible(true);
						    try {
							annotationsValidator.doValidator(field.get(arg), fieldAnn);
						    } catch (CustomValidatorException ex) {
							String errMsg = null;
							if(StringUtil.isNull(ex.getErrorMsg())) {
							    errMsg = ex.getStatusCode().getDefaultMessage();
							} else {
							    errMsg = ex.getErrorMsg();
							}
						        return makeResponse(ex.getStatusCode(), errMsg);
						    } catch (Exception ex) {
							logger.error("验证器【{}】里抛出了 CustomValidatorException 以外的异常,请验证器开发人员注意!!!", ex, fieldAnn.annotationType());
						        return makeResponse(ResponseCodeEnum._500, "服务器内部错误:====" + ex.getMessage());
						    }
						}
					    }
					}
				    }
				} catch (NoSuchFieldException | SecurityException | IllegalArgumentException | IllegalAccessException e) {
				    logger.error("验证器处理切面出了点问题", e);
				}

			    }
			}
		    }
		}
	    }
	}
	Object ret = null;
	try {
	    ret = pjp.proceed();
	} catch (Throwable e) {
	    throw new RuntimeException("AOP Point Cut ValidatorAdvise Throw Exception :", e);
	}
	return ret;
    }
    
    /**
     * @Title: getAllFields
     * @Description: 递归获取该类的所有属性包括父类的爷爷类的...祖宗类的
     * @param fieldList
     * @param classz
     * @return
     */
    private List<Field> getAllFields(List<Field> fieldList, Class<?> classz) {
	if(classz == null) {
	    return fieldList;
	}
	if(fieldList == null) {
	    fieldList = Arrays.asList(classz.getDeclaredFields());  // 获得该类的所有字段,但不包括父类的
	} else {
	    Collections.addAll(fieldList, classz.getDeclaredFields());  // 获得该类的所有字段,但不包括父类的
	}
	return getAllFields(fieldList, classz.getSuperclass());
    }
    
    /**
     * @Title: makeResponse
     * @Description: 生成统一 Response
     * @param statusCode
     * @param statusMessage
     * @return
     */
    private AppApiResponse<?> makeResponse(ResponseCodeEnum statusCode, String statusMessage) {
	AppApiResponse<Object> response = new AppApiResponse<>(new Object());
	AppApiResponseHeader respHeader = new AppApiResponseHeader();
        response.setHeader(respHeader);
        respHeader.setStatusCode(statusCode);
        respHeader.setStatusMessage(statusMessage);
        return response;
    }
    
}
```
# 捕获异常,生成统一格式响应
利用springMVC的@ControllerAdvice捕获所有来自 Controller 的异常
```
@ControllerAdvice(basePackages = "org.test.appApi.actions")
public class ErrorHandlingControllerAdvice {

    private static Logger logger = LoggerFactory.getLogger(ErrorHandlingControllerAdvice.class);
    
    /**
     * @Title: handleValidationError
     * @Description: 处理表单验证,业务异常
     * @param ex
     * @return
     */
    @ExceptionHandler(CustomValidatorException.class)
    @ResponseStatus(HttpStatus.OK)
    @ResponseBody
    public AppApiResponse<?> handleValidationError(CustomValidatorException ex) {
	String errMsg = null;
	if(StringUtil.isNull(ex.getErrorMsg())) {
	    errMsg = ex.getStatusCode().getDefaultMessage();
	} else {
	    errMsg = ex.getErrorMsg();
	}
        return makeResponse(ex.getStatusCode(), errMsg);
    }
    
    /**
     * @Title: handleValidationError
     * @Description: 处理其他异常
     * @param ex
     * @return
     */
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.OK)
    @ResponseBody
    public AppApiResponse<?> handleValidationError(Exception ex) {
	logger.error("服务器内部错误:====", ex);
        return makeResponse(ResponseCodeEnum._500, "服务器内部错误:====" + ex.getMessage());
    }
    
    private AppApiResponse<?> makeResponse(ResponseCodeEnum statusCode, String statusMessage) {
	AppApiResponse<Object> response = new AppApiResponse<>(new Object());
	AppApiResponseHeader respHeader = new AppApiResponseHeader();
        response.setHeader(respHeader);
        respHeader.setStatusCode(statusCode);
        respHeader.setStatusMessage(statusMessage);
        return response;
    }
    
}
```
# 配制自定义验证器切面
springMVC的配制文件中增加
```
<bean id="validatorAdvise" class="org.test.appApi.actions.validator.advises.ValidatorAdvise" />
	<aop:config>
		<aop:aspect id="validatorAop" ref="validatorAdvise">
			<aop:pointcut id="validator" expression="execution(* org.test.appApi.actions..*Action.*(..)) 
			and !execution(* org.test.appApi.actions..*Action.initBinder(..)) 
			and !execution(* org.test.appApi.actions..*Action.set*(..)) 
			and !execution(* org.test.appApi.actions..*Action.get*(..))" />
			<aop:around pointcut-ref="validator" method="validator" />
		</aop:aspect>
	</aop:config>
```
# 使用注解
使用注解很简单,springMVC的控制器中的方法可以定义任意类型的参数,把各种参数放到一个java bean中,并在该bean使用类注解 @EnableValidator开启验证,并在需要验证的类属性上使用对应的验证器注解就行,验证器注解可以多个混合使用
# 写在结束
到这里,自定义验证器的开发就完成了.也许大家有更好的办法,欢迎讨论.也许spring MVC可以有办法实现我想做的但我不知道,也欢迎大家指出.