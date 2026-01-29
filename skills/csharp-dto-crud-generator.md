---
name: "csharp-dto-crud-generator"
description: "根据 C# 实体生成 Create/Update/Detail/List DTO 及 CRUD 服务代码。"
---

# DTO & CRUD 服务类生成器

基于 C# 实体模型 (Entity) 自动生成 DTOs、AutoMapper 映射及 Service 层代码。

## 执行流程

1.  **解析实体**
    - 解析指定实体类及其基类，提取属性、类型及特性注解。

2.  **生成/更新映射表**
    - 在实体同级目录生成或更新 `[EntityName]映射表.md`。
    - **表格规则**：
        - 列：`字段`, `CreateDTO`, `UpdateDTO`, `DetailDTO`, `ListDTO`。
        - 每一行对应一个属性。单元格内容：
            - `✔` (或空): 表示包含/不包含。
            - `[TypeName]`: 指定 DTO 中该属性的特殊类型（覆盖默认类型）。
        - **默认策略**：
            - ID / 审计字段 (`CreatedAt`, `UpdatedAt`): 仅 Detail/List，排除 Create/Update。
            - 导航属性 / `virtual` / `IEnumerable`: 仅 Detail，排除其他。
            - 其他字段: 默认全选。
    - **注意**: 必须等待用户确认映射表内容后，才进行后续代码生成。

3.  **生成代码**
    - **目录规范**: 自动创建缺失目录。
        - DTOs: `项目根目录/Application.Contracts/[EntityName]/`
        - Maps & Services: `项目根目录/Application/[EntityName]/`
    
    - **DTO 生成**:
        - `CreateDTO`/`UpdateDTO`: 基于映射表生成，保留验证特性 (`[Required]`, `[StringLength]` 等)。
        - `DetailDTO`/`ListDTO`: 基于映射表生成。
        - 命名空间和实体相同
    
    - **Map 生成**:
        - 创建 `[EntityName]MapProfile`。
        - 添加从Entity到所有DTO的映射关系。
        - 添加从CreateDTO/UpdateDTO到Entity的映射关系。
        - 命名空间和实体相同
    
    - **Service 生成**:
        - 接口: `I[EntityName]Service : ICurdService<Entity, CreateDto, UpdateDto, DetailDto, ListDto>`
        - 实现: `[EntityName]Service : CurdService<...>, I[EntityName]Service`
        - 包含标准 CRUD 方法 (`GetAsync`, `ListAsync`, `CreateAsync`, `UpdateAsync`, `DeleteAsync`)。
        - 命名空间和实体相同

## 交互示例

**用户**: "为 User 实体生成 DTO"

**助手**:
1. 分析 `User` 类。
```csharp
public class User
{
    [StringLength(100)]
    public string Name { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```
2. 生成 `User映射表.md`:
   ```markdown
   | 字段 | CreateDTO | UpdateDTO | DetailDTO | ListDTO |
   | :--- | :--- | :--- | :--- | :--- |
   | Name | ✔ | ✔ | ✔ | ✔ |
   | CreatedAt | | | ✔ | ✔ |
   | UpdatedAt | | | ✔ | DateTimeOffset |
   ```
   *(注: 用户可修改表格，如将 UpdatedAt 的 ListDTO 类型改为 DateTimeOffset)*

3. 用户确认后，生成文件:
   - `Application.Contracts/User/UserCreateDto.cs`
   ```csharp
    public class UserCreateDto
    {
        [StringLength(100)]
        public string Name { get; set; }
    }
    ```
   - `Application.Contracts/User/UserUpdateDto.cs`
   ```csharp
    public class UserUpdateDto
    {
        [StringLength(100)]
        public string Name { get; set; }
    }
    ```
   - `Application.Contracts/User/UserDetailDto.cs`
   ```csharp
    public class UserDetailDto
    {
        public string Name { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
    }
    ```
   - `Application.Contracts/User/UserListDto.cs`
   ```csharp
    public class UserListDto
    {
        public string Name { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTimeOffset UpdatedAt { get; set; }
    }
    ```
   - `Application/User/UserMapProfile.cs`
   ```csharp
    public class UserMapProfile : Profile
    {
        public UserMapProfile()
        {
            CreateMap<User, UserCreateDto>();
            CreateMap<UserCreateDto, User>();
            CreateMap<User, UserUpdateDto>();
            CreateMap<UserUpdateDto, User>();
            CreateMap<User, UserDetailDto>();
            CreateMap<User, UserListDto>();
        }
    }
    ```
   - `Application/User/IUserService.cs`
    ```csharp
    public interface IUserService: ICurdService<User, UserCreateDto, UserUpdateDto, UserDetailDto, UserListDto>
    {
    }
    ```
   - `Application/User/UserService.cs`
    ```csharp
    public class UserService : CurdService<User, UserCreateDto, UserUpdateDto, UserDetailDto, UserListDto>, IUserService, IScoped
    {
    }
    ```
