## Apifox

1. Apifox 怎么使用

----


## Java 文档

1. Swagger 是什么
2. Swagger 与 OpenAPI 的区别
3. SpringDoc 中常用的三种注解
	1. @Tag、@Schema、@Operation
4. @Tag 是干什么的，有什么属性
	1. 为 Controller 类中所有方法添加标签
	2. name、description
5. @Operation 是干什么的，有什么属性
	1. 为一个方法添加标签
	2. summary、description、tags、parameters（请求参数）、requestBody（请求体）、responses（返回响应）、deprecated（是否弃用）
6. @Schema 是干什么的，有什么属性
	1. name、description、example、defaultValue、required、format、allowableValues（可选值）
7. @Parameter 是干什么的，有什么属性
	1. name、description、example、schema、in、required、allowEmptyValue
8. @RequestBody 是干什么的，有什么属性
	1. description、required、content
9. @ApiResponse 是干什么的，有什么属性
	1. responseCode、description、content
10. @Content 是干嘛的，有什么属性
	1. mediaType（MIME类型）、schema、examples
11. SpringDoc 如何集成 Spring Security

----


## NFS

1. NFS 基本使用流程

---


## Spring Boot

1. 六大请求映射注解
	1. RGDPPP
	2. @RequestMapping、@GetMapping、@DeleteMapping、@PostMapping、@PutMapping、@PatchMapping
2. 六大请求映射注解中的 URI 路径中的 通配符 怎么写？
3. @RestController 是哪些注解组成的？
4. @ResponseBody 是干什么的？
5. @RequestBody 是干什么？
6. @RequestParam 是干的吗？
7. @PathVariable 是干什么？
8. Spring Ioc 六大注解
	1. RRSCCB
	2. @RestController、@Repository、@Service、@Component、@Controller、@Bean

---


## 术语

1. 主从和主备的区别
	1. 主从和主备的区别是，备节点不参与业务处理，只有主节点挂掉时才会切换到备节点继续服务。

----
