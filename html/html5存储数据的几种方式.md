### Html5存储数据的几种方式
#### Cookie
- Cookie（复数形态Cookies），中文名称为“小型文本文件”或“小甜饼”，指某些网站为了辨别用户身份而储存在用户本地终端（Client Side）上的数据（通常经过加密）。
- Cookie总是保存在客户端中，按在客户端中的存储位置，可分为内存Cookie和硬盘Cookie。内存Cookie由浏览器维护，保存在内存中，浏览器关闭后就消失了，其存在时间是短暂的。硬盘Cookie保存在硬盘里，有一个过期时间，除非用户手工清理或到了过期时间，硬盘Cookie不会被删除，其存在时间是长期的。所以，按存在时间，可分为非持久Cookie和持久Cookie。
- 因为HTTP协议是无状态的，即服务器不知道用户上一次做了什么，所以Cookie就是用来绕开HTTP的无状态性的“额外手段”之一。服务器可以设置或读取Cookies中包含的信息，借此维护用户跟服务器会话中的状态。
- 缺陷：Cookie会被附加在每个HTTP请求中，所以无形中增加了流量；
由于在HTTP请求中的Cookie是明文传递的，所以安全性成问题（除非用HTTPS）；
Cookie的大小限制在4KB左右，对于复杂的存储需求来说是不够用的。

#### localStorage
- localStorage对象为源提供了一个Storage对象。并且没有时间限制。客户端必须具有一组本地存储区域，每个存储区域一个。客户端程序只能出于安全原因或用户请求这样做，才能从本地存储区域使数据过期。客户端应始终避免在可以访问该数据的脚本正在运行时删除数据。
- 当访问localStorage属性时，客户端必须运行以下步骤，这些步骤称为存储对象初始化步骤：如果请求违反策略决定（例如，如果客户端被配置为不允许页面保留数据），用户代理可能会抛出SecurityError异常并中止这些步骤，而不是返回Storage对象。如果Document的origin不是scheme/host/port/tuple，那么抛出SecurityError异常并中止这些步骤。检查客户端是否为访问该属性的Window对象的Document的原始文件分配了本地存储区域。如果没有，请创建一个新的存储区域。返回与该源的本地存储区域相关联的存储对象。每个Document对象必须为其Window的localStorage属性设置一个单独的对象。
当在与本地存储区域相关联的Storage对象x上调用setItem（），removeItem（）和clear（）方法时，如果方法没有抛出异常或如上定义的“do nothing”，则Window对象的localStorage属性的Storage对象与x之外的相同存储区域相关联的每个Document对象发送存储通知。
- 无论是在属性枚举期间，当确定存在的属性数量时，无论作为直接属性访问的一部分，在检查，返回，设置还是删除localStorage属性的Storage对象的属性时，或者作为执行存储接口上定义的任何方法或属性的一部分，用户代理必须首先获取存储对象。

#### sessionStorage
- 针对一个session的数据存储。sessionStorage属性表示当前顶层浏览上下文特有的存储区集合。每个顶级浏览上下文(browsing context)都有一组唯一的会话存储区域，每个区域都有一个。客户端不应该从浏览上下文的会话存储区域使数据过期，但是当用户请求删除此类数据时，或者当客户端检测到其存储空间有限时，或出于安全考虑，可能会这样做。客户端应始终避免在可以访问该数据的脚本正在运行时删除数据。当顶级浏览上下文被破坏（并且因此用户永久不可访问）时，存储在其会话存储区域中的数据可以被丢弃。浏览上下文的生命周期可能与实际客户端进程本身的生命周期无关，因为客户端可能会在重新启动后支持恢复会话。当在具有顶级浏览上下文的浏览上下文中创建新文档时，客户端必须检查该顶级浏览上下文是否具有该文档起源的会话存储区域。如果是这样，那就是文档分配的会话存储区域。如果没有，则必须创建该文档原始的新存储区域，然后是文档分配的会话存储区域。文档的分配存储区域在文档的生命周期内不会更改。
- 在将iframe移动到另一个文档的情况下，嵌套的浏览上下文将被破坏，并创建一个新的。
- sessionStorage属性必须返回与文档分配的会话存储区域相关联的Storage对象（如果有），如果没有，则返回null。每个Document对象必须有一个单独的对象，用于其Window的sessionStorage属性。
- 当通过克隆现有的浏览上下文创建新的顶级浏览上下文时，新的浏览上下文必须以与原始的相同的会话存储区域开始，但是从这一点开始，两个集合必须被认为是分开的，而不是以某种方式彼此影响。
- 当现有浏览环境中的脚本创建新的顶级浏览上下文时，或者在现有浏览环境中的用户跟随某个其他方式与特定文档相关的用户创建新的顶级浏览上下文时，创建不是新的开始会话存储，则该文档的起始位置的会话存储区域在创建时必须复制到新的浏览上下文中。然而，从那时起，两个会议存储区域必须被视为分开的，不会以任何方式相互影响。
- 当在与会话存储区域相关联的存储对象x上​​调用setItem（），removeItem（）和clear（）方法时，如果方法没有抛出如上定义的异常或“do nothing”，则Window对象的sessionStorage属性的Storage对象与x之外的相同存储区域相关联的每个Document对象发送存储通知。
