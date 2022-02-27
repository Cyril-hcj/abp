# Web应用程序开发教程 - 第五章: 授权
````json
//[doc-params]
{
    "UI": ["MVC","Blazor","BlazorServer","NG"],
    "DB": ["EF","Mongo"]
}
````
## 关于本教程

在本系列教程中, 你将构建一个名为 `Acme.BookStore` 的用于管理书籍及其作者列表的基于ABP的应用程序.  它是使用以下技术开发的:

* **{{DB_Text}}** 做为ORM提供程序.
* **{{UI_Value}}** 做为UI框架.

本教程分为以下部分:

- [Part 1: 创建服务端](Part-1.md)
- [Part 2: 图书列表页面](Part-2.md)
- [Part 3: 创建,更新和删除图书](Part-3.md)
- [Part 4: 集成测试](Part-4.md)
- **Part 5: 授权**(本章)
- [Part 6: 作者: 领域层](Part-6.md)
- [Part 7: 作者: 数据库集成](Part-7.md)
- [Part 8: 作者: 应用服务层](Part-8.md)
- [Part 9: 作者: 用户页面](Part-9.md)
- [Part 10: 图书到作者的关系](Part-10.md)

### 下载源代码

本教程根据你的**UI** 和 **Database**偏好有多个版,我们准备了两种可供下载的源码组合:

* [MVC (Razor Pages) UI with EF Core](https://github.com/abpframework/abp-samples/tree/master/BookStore-Mvc-EfCore)
* [Blazor UI with EF Core](https://github.com/abpframework/abp-samples/tree/master/BookStore-Blazor-EfCore)
* [Angular UI with MongoDB](https://github.com/abpframework/abp-samples/tree/master/BookStore-Angular-MongoDb)

> 如果您在 Windows 上遇到“文件名太长”或“解压缩错误”，可能与 Windows 最大文件路径限制有关。Windows 的最大文件路径限制为 250 个字符.要解决此问题， [请在 Windows 10 中启用长路径选项](https://docs.microsoft.com/zh-cn/windows/win32/fileio/maximum-file-path-limitation?tabs=cmd#enable-long-paths-in-windows-10-version-1607-and-later).

> 如果您 Git 遇到长路径相关的错误, 请尝试以下命令在 Windows 中启用长路径.见 https://github.com/msysgit/msysgit/wiki/Git-cannot-create-a-file-or-directory-with-a-long-path
> `git config --system core.longpaths true`

{{if UI == "MVC" && DB == "EF"}}

### 视频教程

这部分也被录制为视频教程并 **<a href="https://www.youtube.com/watch?v=1WsfMITN_Jk&list=PLsNclT2aHJcPNaCf7Io3DbMN6yAk_DgWJ&index=5" target="_blank">发布在 YouTube 上</a>**.

{{end}}

## 权限

ABP 框架提供了一个基于 ASP.NET Core 的[授权基础架构](https://docs.microsoft.com/zh-cn/aspnet/core/security/authorization/introduction)的[授权系统](../Authorization.md).在标准授权基础架构之上添加的一项主要功能为**权限系统**，它允许定义权限并启用/禁用每个角色、用户或客户端.

### 权限名称

权限必须具有唯一的名称 (a `string`). 最好的方法是将其定义为 `常数`, 这样我们就可以重用权限名称.

打开项目`Acme.BookStore.Application.Contracts`里的`BookStorePermissions`类（在 `Permissions` 文件夹中）并更改内容如下所示：

````csharp
namespace Acme.BookStore.Permissions
{
    public static class BookStorePermissions
    {
        public const string GroupName = "BookStore";

        public static class Books
        {
            public const string Default = GroupName + ".Books";
            public const string Create = Default + ".Create";
            public const string Edit = Default + ".Edit";
            public const string Delete = Default + ".Delete";
        }
    }
}
````

这是定义权限名称的分层方式。例如, "create book" 权限名称被定义为 `BookStore.Books.Create`. ABP 不会强制您使用此结构，但我们发现这种方式很有用.

### 权限定义

您应该在使用之前定义好权限.

打开项目 `Acme.BookStore.Application.Contracts` 内的 `BookStorePermissionDefinitionProvider` 类（在Permissions文件夹中）并更改内容如下所示：

````csharp
using Acme.BookStore.Localization;
using Volo.Abp.Authorization.Permissions;
using Volo.Abp.Localization;

namespace Acme.BookStore.Permissions
{
    public class BookStorePermissionDefinitionProvider : PermissionDefinitionProvider
    {
        public override void Define(IPermissionDefinitionContext context)
        {
            var bookStoreGroup = context.AddGroup(BookStorePermissions.GroupName, L("Permission:BookStore"));

            var booksPermission = bookStoreGroup.AddPermission(BookStorePermissions.Books.Default, L("Permission:Books"));
            booksPermission.AddChild(BookStorePermissions.Books.Create, L("Permission:Books.Create"));
            booksPermission.AddChild(BookStorePermissions.Books.Edit, L("Permission:Books.Edit"));
            booksPermission.AddChild(BookStorePermissions.Books.Delete, L("Permission:Books.Delete"));
        }

        private static LocalizableString L(string name)
        {
            return LocalizableString.Create<BookStoreResource>(name);
        }
    }
}
````

该类定义了一个 **权限组**（对 UI 上的权限进行分组，将在下面看到）和该组内的**4 个权限**.此外，**Create**, **Edit** 和 **Delete** 是 `BookStorePermissions.Books.Default` 权限的子级. **只有选择了父级** 才能选择子级权限.

最后，编辑本地化文件（`en.json` 在 `Acme.BookStore.Domain.Shared` 项目的 `Localization/BookStore`文件夹下  ）以定义上面使用的本地化键：

````json
"Permission:BookStore": "Book Store",
"Permission:Books": "Book Management",
"Permission:Books.Create": "Creating new books",
"Permission:Books.Edit": "Editing the books",
"Permission:Books.Delete": "Deleting the books"
````

> 本地化键名是任意的，没有强制规则.但我们更喜欢上面使用的约定.

### 权限管理界面

定义权限后，您可以在**权限管理模式**中看到它们.

进入*管理 -> 身份认证管理 -> 角色*页面，为 admin 角色选择*权限*操作以打开权限管理模式：

![bookstore-permissions-ui](images/bookstore-permissions-ui.png)

授予您想要的权限并保存.

> **提示**：如果您运行 `Acme.BookStore.DbMigrator` 应用程序，新权限会自动授予管理员角色.

## 授权

现在，您可以使用权限来授权图书管理.

### 应用层和 HTTP API

打开`BookAppService`类添加策略名称并设置为上面定义的权限名称：

````csharp
using System;
using Acme.BookStore.Permissions;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.BookStore.Books
{
    public class BookAppService :
        CrudAppService<
            Book, //The Book entity
            BookDto, //Used to show books
            Guid, //Primary key of the book entity
            PagedAndSortedResultRequestDto, //Used for paging/sorting
            CreateUpdateBookDto>, //Used to create/update a book
        IBookAppService //implement the IBookAppService
    {
        public BookAppService(IRepository<Book, Guid> repository)
            : base(repository)
        {
            GetPolicyName = BookStorePermissions.Books.Default;
            GetListPolicyName = BookStorePermissions.Books.Default;
            CreatePolicyName = BookStorePermissions.Books.Create;
            UpdatePolicyName = BookStorePermissions.Books.Edit;
            DeletePolicyName = BookStorePermissions.Books.Delete;
        }
    }
}
````

向构造函数添加代码。`CrudAppService` 基类自动在 CRUD 操作上使用这些权限。这使**应用程序服务**安全，但也使**HTTP API**安全，因为该服务自动用作于 HTTP API，如前所述(请参阅[自动 API 控制器](../API/Auto-API-Controllers.md)).

> 稍后在开发作者管理功能时，您将看到使用 `[Authorize(...)]` 特性的声明性授权.

{{if UI == "MVC"}}

### Razor Page

在保护 HTTP API 和应用程序服务防止未经授权的用户使用服务的同时，他们仍然可以导航到图书管理页面。虽然当页面对服务器进行第一次 AJAX 调用时它们会获得授权异常，但我们还应该授权页面以获得更好的用户体验和安全性.

打开 `BookStoreWebModule` 并在 `ConfigureServices` 方法中添加以下代码块：

````csharp
Configure<RazorPagesOptions>(options =>
{
    options.Conventions.AuthorizePage("/Books/Index", BookStorePermissions.Books.Default);
    options.Conventions.AuthorizePage("/Books/CreateModal", BookStorePermissions.Books.Create);
    options.Conventions.AuthorizePage("/Books/EditModal", BookStorePermissions.Books.Edit);
});
````

现在，未经授权的用户被重定向到**登录页面**.

#### 隐藏"New Book"按钮

图书管理页面有一个*New Book*按钮,如果当前用户没有*创建图书*权限,该按钮应该是不可见的.

![bookstore-new-book-button-small](images/bookstore-new-book-button-small.png)

打开 `Pages/Books/Index.cshtml` 文件然后修改内容如下图：

````html
@page
@using Acme.BookStore.Localization
@using Acme.BookStore.Permissions
@using Acme.BookStore.Web.Pages.Books
@using Microsoft.AspNetCore.Authorization
@using Microsoft.Extensions.Localization
@model IndexModel
@inject IStringLocalizer<BookStoreResource> L
@inject IAuthorizationService AuthorizationService
@section scripts
{
    <abp-script src="/Pages/Books/Index.js"/>
}

<abp-card>
    <abp-card-header>
        <abp-row>
            <abp-column size-md="_6">
                <abp-card-title>@L["Books"]</abp-card-title>
            </abp-column>
            <abp-column size-md="_6" class="text-right">
                @if (await AuthorizationService.IsGrantedAsync(BookStorePermissions.Books.Create))
                {
                    <abp-button id="NewBookButton"
                                text="@L["NewBook"].Value"
                                icon="plus"
                                button-type="Primary"/>
                }
            </abp-column>
        </abp-row>
    </abp-card-header>
    <abp-card-body>
        <abp-table striped-rows="true" id="BooksTable"></abp-table>
    </abp-card-body>
</abp-card>
````

* 添加 `@inject IAuthorizationService AuthorizationService` 访问授权服务.
* 使用 `@if (await AuthorizationService.IsGrantedAsync(BookStorePermissions.Books.Create))` 检查图书创建权限以有条件地呈现 *New Book* 按钮.

### JavaScript 端

图书管理页面中的图书表的每一行都有一个操作按钮。操作按钮包括 *Edit* and *Delete* 操作：

![bookstore-edit-delete-actions](images/bookstore-edit-delete-actions.png)

如果当前用户没有授予相关权限，我们应该隐藏该操作. Datatables 行操作有一个 `visible` 选项,如果设置为false可以隐藏该操作项.

打开 `Acme.BookStore.Web` 项目里的 `Pages/Books/Index.js` ,给 `Edit` 操作添加一个`visible` 选项,如下图:

````js
{
    text: l('Edit'),
    visible: abp.auth.isGranted('BookStore.Books.Edit'), //CHECK for the PERMISSION
    action: function (data) {
        editModal.open({ id: data.record.id });
    }
}
````

`Delete` 选项也是一样的操作:

````js
visible: abp.auth.isGranted('BookStore.Books.Delete')
````

* `abp.auth.isGranted(...)` 用于检查之前定义的权限.
* `visible` 也可以是一个返回 `bool` 的函数，如果该值稍后会根据某些条件进行计算.

### 菜单项

即使我们已经保护了图书管理页面的所有层,它仍然在应用程序的主菜单中可见.如果当前用户没有权限，我们应该隐藏菜单项.

打开 `BookStoreMenuContributor` 类, 找到下面的代码块:

````csharp
context.Menu.AddItem(
    new ApplicationMenuItem(
        "BooksStore",
        l["Menu:BookStore"],
        icon: "fa fa-book"
    ).AddItem(
        new ApplicationMenuItem(
            "BooksStore.Books",
            l["Menu:Books"],
            url: "/Books"
        )
    )
);
````

并将此代码块替换为以下内容:

````csharp
var bookStoreMenu = new ApplicationMenuItem(
    "BooksStore",
    l["Menu:BookStore"],
    icon: "fa fa-book"
);

context.Menu.AddItem(bookStoreMenu);

//CHECK the PERMISSION
if (await context.IsGrantedAsync(BookStorePermissions.Books.Default))
{
    bookStoreMenu.AddItem(new ApplicationMenuItem(
        "BooksStore.Books",
        l["Menu:Books"],
        url: "/Books"
    ));
}
````

您还需要添加 `async` 关键字到 `ConfigureMenuAsync` 方法并重新排列返回值.最后 `BookStoreMenuContributor` 类应该如下：

````csharp
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Localization;
using Acme.BookStore.Localization;
using Acme.BookStore.MultiTenancy;
using Acme.BookStore.Permissions;
using Volo.Abp.TenantManagement.Web.Navigation;
using Volo.Abp.UI.Navigation;

namespace Acme.BookStore.Web.Menus
{
    public class BookStoreMenuContributor : IMenuContributor
    {
        public async Task ConfigureMenuAsync(MenuConfigurationContext context)
        {
            if (context.Menu.Name == StandardMenus.Main)
            {
                await ConfigureMainMenuAsync(context);
            }
        }

        private async Task ConfigureMainMenuAsync(MenuConfigurationContext context)
        {
            if (!MultiTenancyConsts.IsEnabled)
            {
                var administration = context.Menu.GetAdministration();
                administration.TryRemoveMenuItem(TenantManagementMenuNames.GroupName);
            }

            var l = context.GetLocalizer<BookStoreResource>();

            context.Menu.Items.Insert(0, new ApplicationMenuItem("BookStore.Home", l["Menu:Home"], "~/"));

            var bookStoreMenu = new ApplicationMenuItem(
                "BooksStore",
                l["Menu:BookStore"],
                icon: "fa fa-book"
            );

            context.Menu.AddItem(bookStoreMenu);

            //CHECK the PERMISSION
            if (await context.IsGrantedAsync(BookStorePermissions.Books.Default))
            {
                bookStoreMenu.AddItem(new ApplicationMenuItem(
                    "BooksStore.Books",
                    l["Menu:Books"],
                    url: "/Books"
                ));
            }
        }
    }
}
````

{{else if UI == "NG"}}

### Angular Guard Configuration

First step of the UI is to prevent unauthorized users to see the "Books" menu item and enter to the book management page.

Open the `/src/app/book/book-routing.module.ts` and replace with the following content:

````js
import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';
import { AuthGuard, PermissionGuard } from '@abp/ng.core';
import { BookComponent } from './book.component';

const routes: Routes = [
  { path: '', component: BookComponent, canActivate: [AuthGuard, PermissionGuard] },
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule],
})
export class BookRoutingModule {}
````

* Imported `AuthGuard` and `PermissionGuard` from the `@abp/ng.core`.
* Added `canActivate: [AuthGuard, PermissionGuard]` to the route definition.

Open the `/src/app/route.provider.ts` and add `requiredPolicy: 'BookStore.Books'` to the `/books` route. The `/books` route block should be following:

````js
{
  path: '/books',
  name: '::Menu:Books',
  parentName: '::Menu:BookStore',
  layout: eLayoutType.application,
  requiredPolicy: 'BookStore.Books',
}
````

### Hide the New Book Button

The book management page has a *New Book* button that should be invisible if the current user has no *Book Creation* permission.

![bookstore-new-book-button-small](images/bookstore-new-book-button-small.png)

Open the `/src/app/book/book.component.html` file and replace the create button HTML content as shown below:

````html
<!-- Add the abpPermission directive -->
<button *abpPermission="'BookStore.Books.Create'" id="create" class="btn btn-primary" type="button" (click)="createBook()">
  <i class="fa fa-plus mr-1"></i>
  <span>{%{{{ '::NewBook' | abpLocalization }}}%}</span>
</button>
````

* Just added `*abpPermission="'BookStore.Books.Create'"` that hides the button if the current user has no permission.

### Hide the Edit and Delete Actions

Books table in the book management page has an actions button for each row. The actions button includes *Edit* and *Delete* actions:

![bookstore-edit-delete-actions](images/bookstore-edit-delete-actions.png)

We should hide an action if the current user has not granted for the related permission.

Open the `/src/app/book/book.component.html` file and replace the edit and delete buttons contents as shown below:

````html
<!-- Add the abpPermission directive -->
<button *abpPermission="'BookStore.Books.Edit'" ngbDropdownItem (click)="editBook(row.id)">
  {%{{{ '::Edit' | abpLocalization }}}%}
</button>

<!-- Add the abpPermission directive -->
<button *abpPermission="'BookStore.Books.Delete'" ngbDropdownItem (click)="delete(row.id)">
  {%{{{ '::Delete' | abpLocalization }}}%}
</button>
````

* Added `*abpPermission="'BookStore.Books.Edit'"` that hides the edit action if the current user has no editing permission.
* Added `*abpPermission="'BookStore.Books.Delete'"` that hides the delete action if the current user has no delete permission.

{{else if UI == "Blazor"}}

### Authorize the Razor Component

Open the `/Pages/Books.razor` file in the `Acme.BookStore.Blazor` project and add an `Authorize` attribute just after the `@page` directive and the following namespace imports (`@using` lines), as shown below:

````html
@page "/books"
@attribute [Authorize(BookStorePermissions.Books.Default)]
@using Acme.BookStore.Permissions
@using Microsoft.AspNetCore.Authorization
...
````

Adding this attribute prevents to enter this page if the current hasn't logged in or hasn't granted for the given permission. In case of attempt, the user is redirected to the login page.

### Show/Hide the Actions

The book management page has a *New Book* button and *Edit* and *Delete* actions for each book. We should hide these buttons/actions if the current user has not granted for the related permissions.

The base `AbpCrudPageBase` class already has the necessary functionality for these kind of operations.

#### Set the Policy (Permission) Names

Add the following code block to the end of the `Books.razor` file:

````csharp
@code
{
    public Books() // Constructor
    {
        CreatePolicyName = BookStorePermissions.Books.Create;
        UpdatePolicyName = BookStorePermissions.Books.Edit;
        DeletePolicyName = BookStorePermissions.Books.Delete;
    }
}
````

The base `AbpCrudPageBase` class automatically checks these permissions on the related operations. It also defines the given properties for us if we need to check them manually:

* `HasCreatePermission`: True, if the current user has permission to create the entity.
* `HasUpdatePermission`: True, if the current user has permission to edit/update the entity.
* `HasDeletePermission`: True, if the current user has permission to delete the entity.

> **Blazor Tip**: While adding the C# code into a `@code` block is fine for small code parts, it is suggested to use the code behind approach to develop a more maintainable code base when the code block becomes longer. We will use this approach for the authors part.

#### Hide the New Book Button

Wrap the *New Book* button by an `if` block as shown below:

````xml
@if (HasCreatePermission)
{
    <Button Color="Color.Primary"
            Clicked="OpenCreateModalAsync">@L["NewBook"]</Button>
}
````

#### Hide the Edit/Delete Actions

`EntityAction` component defines `Visible` attribute (parameter) to conditionally show the action.

Update the `EntityActions` section as shown below:

````xml
<EntityActions TItem="BookDto" EntityActionsColumn="@EntityActionsColumn">
    <EntityAction TItem="BookDto"
                  Text="@L["Edit"]"
                  Visible=HasUpdatePermission
                  Clicked="() => OpenEditModalAsync(context)" />
    <EntityAction TItem="BookDto"
                  Text="@L["Delete"]"
                  Visible=HasDeletePermission
                  Clicked="() => DeleteEntityAsync(context)"
                  ConfirmationMessage="()=>GetDeleteConfirmationMessage(context)" />
</EntityActions>
````

#### About the Permission Caching

You can run and test the permissions. Remove a book related permission from the admin role to see the related button/action disappears from the UI.

**ABP Framework caches the permissions** of the current user in the client side. So, when you change a permission for yourself, you need to manually **refresh the page** to take the effect. If you don't refresh and try to use the prohibited action you get an HTTP 403 (forbidden) response from the server.

> Changing a permission for a role or user immediately available on the server side. So, this cache system doesn't cause any security problem.

### Menu Item

Even we have secured all the layers of the book management page, it is still visible on the main menu of the application. We should hide the menu item if the current user has no permission.

Open the `BookStoreMenuContributor` class in the `Acme.BookStore.Blazor` project, find the code block below:

````csharp
context.Menu.AddItem(
    new ApplicationMenuItem(
        "BooksStore",
        l["Menu:BookStore"],
        icon: "fa fa-book"
    ).AddItem(
        new ApplicationMenuItem(
            "BooksStore.Books",
            l["Menu:Books"],
            url: "/books"
        )
    )
);
````

And replace this code block with the following:

````csharp
var bookStoreMenu = new ApplicationMenuItem(
    "BooksStore",
    l["Menu:BookStore"],
    icon: "fa fa-book"
);

context.Menu.AddItem(bookStoreMenu);

//CHECK the PERMISSION
if (await context.IsGrantedAsync(BookStorePermissions.Books.Default))
{
    bookStoreMenu.AddItem(new ApplicationMenuItem(
        "BooksStore.Books",
        l["Menu:Books"],
        url: "/books"
    ));
}
````

You also need to add `async` keyword to the `ConfigureMenuAsync` method and re-arrange the return value. The final `ConfigureMainMenuAsync` method should be the following:

````csharp
private async Task ConfigureMainMenuAsync(MenuConfigurationContext context)
{
    var l = context.GetLocalizer<BookStoreResource>();

    context.Menu.Items.Insert(
        0,
        new ApplicationMenuItem(
            "BookStore.Home",
            l["Menu:Home"],
            "/",
            icon: "fas fa-home"
        )
    );

    var bookStoreMenu = new ApplicationMenuItem(
        "BooksStore",
        l["Menu:BookStore"],
        icon: "fa fa-book"
    );

    context.Menu.AddItem(bookStoreMenu);

    //CHECK the PERMISSION
    if (await context.IsGrantedAsync(BookStorePermissions.Books.Default))
    {
        bookStoreMenu.AddItem(new ApplicationMenuItem(
            "BooksStore.Books",
            l["Menu:Books"],
            url: "/books"
        ));
    }
}
````

{{end}}

## The Next Part

See the [next part](Part-6.md) of this tutorial.
