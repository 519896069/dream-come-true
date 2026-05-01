# 单元测试规范

## 测试原则

1. **每个测试函数只测一件事**
2. **Arrange-Act-Assert 模式**
3. **测试数据要独立**
4. **测试名称要描述意图**

## 测试结构

```php
class UserServiceTest extends TestCase
{
    public function testCreateUser_Success()
    {
        // Arrange
        $data = ['name' => 'test', 'email' => 'test@example.com'];

        // Act
        $result = $this->service->create($data);

        // Assert
        $this->assertNotNull($result->id);
    }
}
```

## 覆盖率要求

- 核心业务逻辑覆盖率 > 80%
- 公共方法覆盖率 > 60%
