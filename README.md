

# Fluent Architecture
#### Uma arquitetura que pretende simplificar a forma como se escreve c�digos para o dia-a-dia
Ferramentas necess�rias:
- Visual Studio 2017/2019 ou Visual Studio Code

---

### 1� Passo:
Crie uma aplica��o web por meio do Visual Studio.
Os seguintes frameworks s�o compat�veis:
 - **Net Framework 4.6.1**
 - **Net Core 2.1**
 - **Net Core 2.2**

### 2� Passo:
Adicione a referencia do pacote ao seu projeto por meio do gerenciador de pacotes do nuget

```nuget
Install-Package Fluent.Architecture.Core
Install-Package Fluent.Architecture.EntityFramework
```
Escolha os pacotes abaixo de acordo com os bancos de dados a serem usados na aplica��o:
```nuget
Install-Package Fluent.Architecture.EntityFramework.MySQL
Install-Package Fluent.Architecture.EntityFramework.Oracle
Install-Package Fluent.Architecture.EntityFramework.PostgreSQL
Install-Package Fluent.Architecture.EntityFramework.SqlServer
```

### 3� Passo:
Crie uma classe que ser� respons�vel pela inicializa��o da arquitetura, conforme exemplo abaixo:
```C#
using Fluent.Architecture;
using Fluent.Architecture.EntityFramework;
using Fluent.Architecture.EntityFramework.PostgreSQL;
using System;

using Fluent.Architecture;
using Fluent.Architecture.EntityFramework;
using Fluent.Architecture.EntityFramework.MySQL;
using System;

public class ArchitectureInit
{
    public static void Setup(IServiceProvider serviceProvider)
    {
        Fluent.Architecture.Setup
        .Init()
        .SetServiceProvider(serviceProvider)
        .UseEntityFramework()
        .AddConnectionString("Server=localhost;Database=testDb;Uid=root;Pwd=admin;", createDatabaseIfNotExists: true, typeof(EfContextMySQL))
        .Build()
        .Run();
    }
}
```

### 4� Passo:
� partir do Startup de sua aplica��o, inicialize a arquitetura:

#### Exemplo no .Net Core
```C#
...
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseMvc();

    Inicializacao.Inicialize(app.ApplicationServices); //A inicializa��o da arquitetura
}
...
```
#### Exemplo no .Net Framework
```C#

public class WebApiApplication : System.Web.HttpApplication
{
    protected void Application_Start()
    {
        Inicializacao.Inicialize(null); //A inicializa��o da arquitetura
    }
}
```

### 5� Passo:
Crie uma entidade:
```C#
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using Fluent.Architecture.Attributes;
using Fluent.Architecture.Entities;
using Fluent.Architecture.EntityFramework;

[Table("Users"), DbType(FluentDbType.MYSQL)]
public class User : FluentEntity
{
    [Key]
    public int Code { get; set; }

    [FluentUniqueKey]
    public string Email { get; set; }

    public string Name { get; set; }
}
```
> Note que usamos o atributo _Table_ para indicar o nome da tabela do banco de dados e o atributo _DbType_ para indicar o tipo de banco de dados a ser usado por essa entidade.
> Uma entidade sempre deve ser criada para um tipo de banco de dados espec�fico. Em casos em que se deseja criar um Model e n�o uma entidade, deve se tratar de forma um pouco diferente. Veja mais em:  [Entidade](Entidade)

### 6� passo:
Crie um controller  conforme abaixo:
```C++
using Fluent.Architecture.Controllers;
using Microsoft.AspNetCore.Mvc;

[Route("api/[Controller]")]
public class UserController : FluentController<User>
{
    // GET api/controller?code=123
    [HttpGet]
    public User Find([FromQuery] User entity)
    {
        return Service.Find(entity);
    }

    // GET api/controller/Count
    [HttpGet("Count")]
    public object Count()
    {
        return Service.Count();
    }

    // POST api/controller
    [HttpPost]
    public User Add([FromBody] User entity)
    {
        return Service.Add(entity);
    }

    // PUT api/controller
    [HttpPut]
    public User Update([FromBody] User entity)
    {
        return Service.Update(entity);
    }

    // DELETE api/controller
    [HttpDelete]
    public User Remove([FromBody] User entity)
    {
        return Service.Remove(entity);
    }

    // DELETE api/controller/RemoveRange
    [HttpDelete("RemoveRange")]
    public void RemoveRange([FromBody] User[] entidades)
    {
        Service.RemoveRange(entidades);
    }
}
```

Todos os m�todos do controller est�o prontos para serem executados.
Execute a aplica��o e teste os m�todos.
Baixe o template o template do link abaixo e importe no seu postman para facilitar os testes.
[Template para importa��o](https://github.com/dn32/Fluent.Architecture/blob/master/Fluent.Architecture.Sample.API/postman/postman-user-v1.json)

### 7� Passo:
Crie uma especifica��o de consulta
```C#
using System;
using System.Linq;
using Fluent.Architecture.Specifications;

public class UserByEmailSpec : FluentSpecification<User>
{
    public string Email { get; set; }

    public UserByEmailSpec AddParameter(string email)
    {
        Email = email;
        return this;
    }

    public override IQueryable<User> Where(IQueryable<User> query)
    {
        return query.Where(x => x.Email.Equals(Email, StringComparison.InvariantCultureIgnoreCase));
    }

    public override IOrderedQueryable<User> Order(IQueryable<User> query)
    {
        return query.OrderBy(x => x.Name);
    }
}
```
> Uma especifica��o define um modelo de consulta e pode ser utilizada em v�rios m�todos do FluentService<>.

### 8� passo:
Adicione esse novo m�todo ao seu controller:
```C#
[HttpGet("GetUserByEMail")]
public User GetUserByEMail(string email)
{
    var spec = CreateSpec<UserByEmailSpec>().AddParameter(email);
    return Service.FirstOrDefault(spec);
}
```
Execute a aplica��o e acesse o endere�o da aplica��o.
Exemplo: http://localhost:5000/api/user/GetUserByEMail?email=dn@dn32.com.br

---

### 9� passo:
Adicione o filtro **FluentExceptionHandlerAttribute** de controle de exce��es � classe FilterConfig ou equivalente, ficando como a seguir:

#### Exemplo no .Net Core
```C#
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc(options => options.Filters.Add(new FluentExceptionHandlerAttribute()));
}
```

#### Exemplo no .Net Framework
```C#
using Fluent.Architecture.Filters;

public class FilterConfig
{
    public static void RegisterGlobalFilters(GlobalFilterCollection filters)
    {
        filters.Add(new HandleErrorAttribute());
        filters.Add(new FluentExceptionHandlerAttribute());
    }
}
```
Eventuais erros agora ser�o apresentados da seguinte forma:
```C++
{
    Message:  "An entity with any of these keys already exists in the database: {Id:0}, {Email:'max@mail.com'}",
    ValidationError:  true
}
```
O erro de valida��o refere-se ao e-mail repetido no banco de dados. Quando decoramos a propriedade Email de User com **FluentUniqueKey**, informamos para o sistema que n�o � permitido duplica��o do valor desse campo.

Veja mais sobre valida��o em [Valida��o](Valida��o).

### Est� tudo pronto para iniciar o trabalho.
---
N�o � obrigat�ria a cria��o de mais nenhum arquivo para o funcionamento b�sico da entidade *User*, mas se for necess�rio tratar uma regra de valida��o, regra de neg�cio, ou algo mais espec�fico, faz-se necess�rio criar itens customizador.

� necess�rio entender cada ponto da arquitetura para executar com sucesso os procedimentos necess�rios para o desenvolvimento de um sistema limpo e bem feito com ela.
Vamos fazer uma abordagem mais detalhada nos pr�ximos t�picos.

Sempre no rodap� da p�gina ser� apresentado o item que deve ser visto ap�s o atual para facilitar a sequ�ncia de atendimento.

Para facilitar o fluxo de aprendizado, sempre que uma informa��o n�o necess�ria para o entendimento geral da arquitetura, esse ser� marcado como Refer�ncia.
Veja um exemplo abaixo:

> **Refer�ncia**:
> Informa��es de refer�ncia s�o para consulta em momentos de necessidade e n�o para aprendizado imediato.

---
Pra prosseguir, veja o item [Entidade](Entidade)