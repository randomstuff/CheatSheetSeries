# DotNet Security Cheat Sheet

## Introduction

This page provides basic .NET security tips for developers.

## Security Announcements

Receive security notifications by selecting the "Watch" button at the following repositories:

- [.NET Core Security Announcements](https://github.com/dotnet/announcements/issues?q=is%3Aopen+is%3Aissue+label%3ASecurity)
- [ASP.NET Core & Entity Framework Core Security Announcements](https://github.com/aspnet/Announcements/issues?q=is%3Aopen+is%3Aissue+label%3ASecurity)

## .NET General Guidance

### Cryptography

#### Encryption for storage

- Use the [Windows Data Protection API (DPAPI)](https://docs.microsoft.com/en-us/dotnet/standard/security/how-to-use-data-protection) for secure local storage of sensitive data.
- Where DPAPI cannot be used, follow the algorithm guidance in the [OWASP Cryptographic Storage Cheat Sheet](Cryptographic_Storage_Cheat_Sheet.md#algorithms).

### Injections

#### SQL Injection

See the [SQL Injection Prevention Cheat Sheet](SQL_Injection_Prevention_Cheat_Sheet) and the [Query Parameterization Cheat Sheet](Query_Parameterization_Cheat_Sheet.md).

DO: Use [Parameterized SQL](https://docs.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlcommand.prepare?view=netframework-4.7.2).

Remember that building your own ad hoc queries in Entity Framework is just as susceptible to SQLi as a plain SQL query. You must parameterized queries as well:

```csharp
var sql = @"Update [User] SET FirstName = @FirstName WHERE Id = @Id";
context.Database.ExecuteSqlCommand(
    sql,
    new SqlParameter("@FirstName", firstname),
    new SqlParameter("@Id", id));
```

#### OS Command Injection

See the [OS Command Injection Defense Cheat Sheet](OS_Command_Injection_Defense_Cheat_Sheet.md).

DO: Use [`ProcessStartInfo.ArgumentList`](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.processstartinfo.argumentlist) instead of `ProcessStartInfo.Arguments`. The former automatically handles argument without having to manually escape special characters.

DO: Validate any dynamic value to make sure it cannot be interpreted as unexpected options by the called program.

Example of suitable usage:

``` csharp
if (!url.StartsWith("https://"))
    throw new ArgumentException("Invalid URL");
Process cmd = new Process();
cmd.StartInfo.FileName = "wget.exe";
cmd.StartInfo.ArgumentList.Add(url);
cmd.Start();
```

DO NOT: execute program using `ProcessStartInfo.Arguments` with dynamic input. This requires    argument escaping which is error-prone and depends on the operating-system.

Example of implementation vulnerable to OS

``` csharp
if (!url.StartsWith("https://"))
    throw new ArgumentException("Invalid URL");
Process cmd = new Process();
cmd.StartInfo.FileName = "wget.exe";
cmd.StartInfo.Arguments = url;
cmd.Start();
```

DO NOT: call command interpreters such as `cmd.exe` or `powershell.exe`. While it is possible to do it safely, it is very easy to trigger OS command injection vulnerability this way.

Example of implementation vulnerable to OS command injection:

``` csharp
if (!url.StartsWith("https://"))
    throw new ArgumentException("Invalid URL");
Process cmd = new Process();
cmd.StartInfo.FileName = "cmd.exe";
cmd.StartInfo.ArgumentList.Add("/c");
cmd.StartInfo.ArgumentList.Add("wget.exe " + url);
cmd.Start();
```

### Serialization

DO NOT: Use the BinaryFormatter type which is dangerous and [not recommended](https://learn.microsoft.com/en-us/dotnet/standard/serialization/binaryformatter-security-guide) for data processing.
.NET offers several in-box serializers that can handle untrusted data safely:

- XmlSerializer and DataContractSerializer to serialize object graphs into and from XML. Do not confuse DataContractSerializer with NetDataContractSerializer.
- BinaryReader and BinaryWriter for reading and writing primitive data types (such as Int32, Double, Boolean, and String) in binary format.
- The System.Text.Json APIs to serialize object graphs into JSON.

### Identification and Authentication Failures

DO: Use [ASP.NET Core Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-2.2&).
ASP.NET Core Identity framework is well configured by default, where it uses secure password hashes and an individual salt. Identity uses the PBKDF2 hashing function for passwords, and generates a random salt per user.

DO: Configure password length and composition settings according to [NIST’s password guidance](https://pages.nist.gov/800-63-4/sp800-63b/authenticators/#passwordver): at least 15 characters when passwords can be used without multifactor authentication (MFA), or at least eight when only used with MFA, without mandatory character-composition rules.

This ASP.NET Core Identity example assumes passwords can be used without MFA. Identity’s [default password validator](https://github.com/dotnet/aspnetcore/blob/215a587e52efa710de84138b0a3374b860b924d8/src/Identity/Extensions.Core/src/PasswordValidator.cs) measures length in UTF-16 code units, so add Unicode code-point length validation to enforce NIST’s minimum. These settings also do not implement breached-password screening; follow the [Authentication Cheat Sheet](Authentication_Cheat_Sheet.md#implement-proper-password-strength-controls) for the complete policy.

``` csharp
//Startup.cs
services.Configure<IdentityOptions>(options =>
{
 // Password settings
 options.Password.RequireDigit = false;
 options.Password.RequiredLength = 15;
 options.Password.RequireNonAlphanumeric = false;
 options.Password.RequireUppercase = false;
 options.Password.RequireLowercase = false;
 options.Password.RequiredUniqueChars = 1;

 options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(30);
 options.Lockout.MaxFailedAccessAttempts = 3;

 options.SignIn.RequireConfirmedEmail = true;

 options.User.RequireUniqueEmail = true;
});
```

DO: Set a cookie policy

e.g

``` csharp
//Startup.cs
services.ConfigureApplicationCookie(options =>
{
 options.Cookie.HttpOnly = true;
 options.ExpireTimeSpan = TimeSpan.FromMinutes(60);
 // See the discussion in A01 for trade-offs on using sliding expiration.
 options.SlidingExpiration = false;
});
```

Below are vulnerabilities that were included in the 2013 or 2017 OWASP Top 10 list
that were not included in the 2021 list. These vulnerabilities are still relevant
but were not included in the 2021 list because they have become less prevalent.

### XML

XXE attacks occur when an XML parse does not properly process user input that contains external entity declarations in the doctype of an XML payload.

[This article](https://docs.microsoft.com/en-us/dotnet/standard/data/xml/xml-processing-options) discusses the most common XML Processing Options for .NET.

Please refer to the [XXE cheat sheet](XML_External_Entity_Prevention_Cheat_Sheet.md#net) for more detailed information on preventing XXE and other XML Denial of Service attacks.

### Cross-Site Scripting (XSS)

DO NOT: Trust any data the user sends you. Prefer allowlists (always safe) over denylists.

You get encoding of all HTML content with MVC3. To properly encode all content whether HTML,
JavaScript, CSS, LDAP, etc., use the Microsoft AntiXSS library:

`Install-Package AntiXSS`

Then set in config:

```xml
<system.web>
<httpRuntime targetFramework="4.5"
enableVersionHeader="false"
encoderType="Microsoft.Security.Application.AntiXssEncoder, AntiXssLibrary"
maxRequestLength="4096" />
```

DO NOT: Use the `[AllowHTML]` attribute or helper class `@Html.Raw` unless you are absolutely
sure that the content you are writing to the browser is safe and has been escaped properly.

DO: Enable a [Content Security Policy](Content_Security_Policy_Cheat_Sheet.md#context). This will prevent your pages from accessing assets they should not be able to access (e.g. malicious scripts):

```xml
<system.webServer>
    <httpProtocol>
        <customHeaders>
            <add name="Content-Security-Policy"
                value="default-src 'none'; style-src 'self'; img-src 'self';
                font-src 'self'; script-src 'self'" />
```

More information can be found in the [Cross Site Scripting Prevention Cheat Sheet](Cross_Site_Scripting_Prevention_Cheat_Sheet.md).

### Unvalidated redirects and forwards

A protection against this was introduced in MVC 3 template. Here is the code:

```csharp
public async Task<ActionResult> LogOn(LogOnViewModel model, string returnUrl)
{
    if (ModelState.IsValid)
    {
        var logonResult = await _userManager.TryLogOnAsync(model.UserName, model.Password);
        if (logonResult.Success)
        {
            await _userManager.LogOnAsync(logonResult.UserName, model.RememberMe);  
            return RedirectToLocal(returnUrl);
...
```

```csharp
private ActionResult RedirectToLocal(string returnUrl)
{
    if (Url.IsLocalUrl(returnUrl))
    {
        return Redirect(returnUrl);
    }
    else
    {
        return RedirectToAction("Landing", "Account");
    }
}
```

### Database

When using SQL Server, prefer [integrated authentication](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication?view=sql-server-ver16) over [SQL authentication](https://learn.microsoft.com/en-us/sql/relational-databases/security/choose-an-authentication-mode?view=sql-server-ver16#connecting-through-sql-server-authentication).

## ASP.NET Guidance

### HTTPS

Redirect a request made over HTTP to HTTPS.

Foe example in `Configure()` of `Startup.cs`:

``` csharp
app.UseHttpsRedirection();
```

#### Debug and Stack Trace

Ensure debug and trace are off in production. This can be enforced using web.config transforms:

```xml
<compilation xdt:Transform="RemoveAttributes(debug)" />
<trace enabled="false" xdt:Transform="Replace"/>
```

### HTTP services

You can configure the HTTP headers in `web.config`:

```xml
<system.web>
    <httpRuntime enableVersionHeader="false"/>
</system.web>
<system.webServer>
    <security>
        <requestFiltering removeServerHeader="true" />
    </security>
    <httpProtocol>
        <customHeaders>
            <add name="X-Content-Type-Options" value="nosniff" />
            <add name="X-Frame-Options" value="DENY" />
            <add name="X-Permitted-Cross-Domain-Policies" value="master-only"/>
            <add name="X-XSS-Protection" value="0"/>
            <remove name="X-Powered-By"/>
        </customHeaders>
    </httpProtocol>
</system.webServer>
```

Or in `Startup.cs`:

``` csharp
app.UseHsts(hsts => hsts.MaxAge(365).IncludeSubdomains());
app.UseXContentTypeOptions();
app.UseReferrerPolicy(opts => opts.NoReferrer());
app.UseXXssProtection(options => options.FilterDisabled());
app.UseXfo(options => options.Deny());

app.UseCsp(opts => opts
 .BlockAllMixedContent()
 .StyleSources(s => s.Self())
 .StyleSources(s => s.UnsafeInline())
 .FontSources(s => s.Self())
 .FormActions(s => s.Self())
 .FrameAncestors(s => s.Self())
 .ImageSources(s => s.Self())
 .ScriptSources(s => s.Self())
 );
```

More information about headers can be found at the [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/).

#### Weak Account management

Ensure cookies are sent with the HttpOnly flag set to prevent client side scripts from accessing the cookie:

```csharp
CookieHttpOnly = true,
```

Reduce the time period a session can be stolen in by reducing session timeout and removing sliding expiration:

The decision to use sliding expiration depends on your application's threat model. Setting `SlidingExpiration` to `false` enforces an absolute session lifetime, which limits how long a stolen session can be reused, at the cost of reduced usability for long-lived interactive sessions. For some applications, enabling sliding expiration (`true`) may be preferred for user experience, as it keeps the session alive as long as the user is active. This convenience comes with increased risk if a session is compromised.

```csharp
ExpireTimeSpan = TimeSpan.FromMinutes(60),
SlidingExpiration = false
```

See [here](https://github.com/johnstaveley/SecurityEssentials/blob/master/SecurityEssentials/App_Start/Startup.Auth.cs) for an example of a full startup code snippet.

Ensure cookies are sent over HTTPS in production. This should be enforced in the config transforms:

```xml
<httpCookies requireSSL="true" />
<authentication>
    <forms requireSSL="true" />
</authentication>
```

Protect LogOn, Registration and password reset methods against brute force attacks by throttling requests (see code below). Consider also using ReCaptcha.

```csharp
[HttpPost]
[AllowAnonymous]
[ValidateAntiForgeryToken]
[AllowXRequestsEveryXSecondsAttribute(Name = "LogOn",
Message = "You have performed this action more than {x} times in the last {n} seconds.",
Requests = 3, Seconds = 60)]
public async Task<ActionResult> LogOn(LogOnViewModel model, string returnUrl)
```

DO NOT: Roll your own authentication or session management. Use the one provided by .NET.

DO NOT: Tell someone if the account exists on LogOn, Registration or Password reset. Say something like 'Either the username or password was incorrect', or 'If this account exists then a reset token will be sent to the registered email address'. This protects against account enumeration.

The feedback to the user should be identical whether or not the account exists, both in terms of content and behavior. E.g., if the response takes 50% longer when the account is real then membership information can be guessed and tested.

### Authorization

DO: Authorize users on all externally facing endpoints. The .NET framework has many ways to authorize a user, use them at method level:

```csharp
[Authorize(Roles = "Admin")]
[HttpGet]
public ActionResult Index(int page = 1)
```

or better yet, at controller level:

```csharp
[Authorize]
public class UserController
```

You can also check roles in code using identity features in .net: `System.Web.Security.Roles.IsUserInRole(userName, roleName)`

You can find more information in the [Authorization Cheat Sheet](Authorization_Cheat_Sheet.md) and
[Authorization Testing Automation Cheat Sheet](Authorization_Testing_Automation_Cheat_Sheet.md).

When you have a resource (object) which can be accessed by a reference (in the sample below this is the `id`), you need to ensure that the user is intended to have access to that resource.

```csharp
// Insecure
public ActionResult Edit(int id)
{
  var user = _context.Users.FirstOrDefault(e => e.Id == id);
  return View("Details", new UserViewModel(user);
}

// Secure
public ActionResult Edit(int id)
{
  var user = _context.Users.FirstOrDefault(e => e.Id == id);
  // Establish user has right to edit the details
  if (user.Id != _userIdentity.GetUserId())
  {
        HandleErrorInfo error = new HandleErrorInfo(
            new Exception("INFO: You do not have permission to edit these details"));
        return View("Error", error);
  }
  return View("Edit", new UserViewModel(user);
}
```

More information can be found in the [Insecure Direct Object Reference Prevention Cheat Sheet](Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.md).

### Password storage

For new ASP.NET Core applications, the `PasswordHasher<TUser>` API provides password hashing functionality. However, [it uses PBKDF2 with HMAC-SHA512](https://github.com/dotnet/dotnet/blob/b0f34d51fccc69fd334253924abd8d6853fad7aa/src/aspnetcore/src/Identity/Extensions.Core/src/PasswordHasher.cs/) (for ASP.NET Core Identity Version 3). It is recommended to use [a stronger method](https://www.latacora.com/blog/post-quantum-cryptographic-right-answers/#password-handling)

Refer to the [Password Storage Cheat Sheet](Password_Storage_Cheat_Sheet.md) for more information.

### Cross-site request forgery

DO NOT: Send sensitive data without validating Anti-Forgery-Tokens ([.NET](https://docs.microsoft.com/en-us/aspnet/web-api/overview/security/preventing-cross-site-request-forgery-csrf-attacks) / [.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/anti-request-forgery?view=aspnetcore-7.0#aspnet-core-antiforgery-configuration)).

DO: Send the anti-forgery token with every POST/PUT request:

#### Using .NET Framework

```csharp
using (Html.BeginForm("LogOff", "Account", FormMethod.Post, new { id = "logoutForm",
                        @class = "pull-right" }))
{
    @Html.AntiForgeryToken()
    <ul class="nav nav-pills">
        <li role="presentation">
        Logged on as @User.Identity.Name
        </li>
        <li role="presentation">
        <a href="javascript:document.getElementById('logoutForm').submit()">Log off</a>
        </li>
    </ul>
}
```

Then validate it at the method or preferably the controller level:

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public ActionResult LogOff()
```

Make sure the tokens are removed completely for invalidation on logout.

```csharp
/// <summary>
/// SECURE: Remove any remaining cookies including Anti-CSRF cookie
/// </summary>
public void RemoveAntiForgeryCookie(Controller controller)
{
    string[] allCookies = controller.Request.Cookies.AllKeys;
    foreach (string cookie in allCookies)
    {
        if (controller.Response.Cookies[cookie] != null &&
            cookie == "__RequestVerificationToken")
        {
            controller.Response.Cookies[cookie].Expires = DateTime.Now.AddDays(-1);
        }
    }
}
```

#### Using .NET Core 2.0 or later

Starting with .NET Core 2.0 it is possible to [automatically generate and verify the antiforgery token](https://docs.microsoft.com/en-us/aspnet/core/security/anti-request-forgery?view=aspnetcore-7.0#aspnet-core-antiforgery-configuration).

With [FormTagHelper enabled](https://learn.microsoft.com/en-us/aspnet/core/security/anti-request-forgery?view=aspnetcore-7.0#aspnet-core-antiforgery-configuration), a form with `method="post"` and an absent or empty `action` automatically gets an antiforgery token unless token generation is disabled. This does not apply to every form. You can check whether tag helpers are enabled in your main `_ViewImports.cshtml` file:

```csharp
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

`IHtmlHelper.BeginForm` generates an antiforgery token by default for methods other than GET. Token generation must be paired with server-side validation, as described below.

For forms that need antiforgery protection without automatic token generation, add the token explicitly:

```html
<form action="RelevantAction" >
@Html.AntiForgeryToken()
</form>
```

To automatically validate all requests other than GET, HEAD, OPTIONS and TRACE you need to add a global action filter with the [AutoValidateAntiforgeryToken](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.autovalidateantiforgerytokenattribute?view=aspnetcore-7.0) attribute inside your `Startup.cs` as mentioned in the following [article](https://andrewlock.net/automatically-validating-anti-forgery-tokens-in-asp-net-core-with-the-autovalidateantiforgerytokenattribute/):

```csharp
services.AddMvc(options =>
{
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute());
});
```

If you need to disable the attribute validation for a specific method on a controller you can add the [IgnoreAntiforgeryToken](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.ignoreantiforgerytokenattribute?view=aspnetcore-7.0) attribute to the controller method (for MVC controllers) or parent class (for Razor pages):

```csharp
[IgnoreAntiforgeryToken]
[HttpDelete]
public IActionResult Delete()
```

```csharp
[IgnoreAntiforgeryToken]
public class UnsafeModel : PageModel
```

If you need to also validate the token on GET, HEAD, OPTIONS and TRACE requests you can add the [ValidateAntiforgeryToken](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.validateantiforgerytokenattribute?view=aspnetcore-7.0) attribute to the controller method (for MVC controllers) or parent class (for Razor pages):

```csharp
[HttpGet]
[ValidateAntiforgeryToken]
public IActionResult DoSomethingDangerous()
```

```csharp
[HttpGet]
[ValidateAntiforgeryToken]
public class SafeModel : PageModel
```

In case you can't use a global action filter, add the [AutoValidateAntiforgeryToken](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.autovalidateantiforgerytokenattribute?view=aspnetcore-7.0) attribute to your controller classes or razor page models:

```csharp
[AutoValidateAntiforgeryToken]
public class UserController
```

```csharp
[AutoValidateAntiforgeryToken]
public class SafeModel : PageModel
```

#### Using .Net Core or .NET Framework with AJAX

You will need to attach the anti-forgery token to AJAX requests.

If you are using jQuery in an ASP.NET Core MVC view this can be achieved using this snippet:

```javascript
@inject  Microsoft.AspNetCore.Antiforgery.IAntiforgery antiforgeryProvider
$.ajax(
{
    type: "POST",
    url: '@Url.Action("Action", "Controller")',
    contentType: "application/x-www-form-urlencoded; charset=utf-8",
    data: {
        id: id,
        '__RequestVerificationToken': '@antiforgeryProvider.GetAndStoreTokens(this.Context).RequestToken'
    }
})
```

If you are using the .NET Framework, you can find some code snippets [here](https://docs.microsoft.com/en-us/aspnet/web-api/overview/security/preventing-cross-site-request-forgery-csrf-attacks#anti-csrf-and-ajax).

More information can be found in the [Cross-Site Request Forgery Prevention Cheat Sheet](Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.md).

## ASP NET Web Forms Guidance

ASP.NET Web Forms is the original browser-based application development API for the .NET Framework, and is still the most common enterprise platform for web application development.

- Always use [HTTPS](http://support.microsoft.com/kb/324069).
- Enable [requireSSL](https://docs.microsoft.com/en-us/dotnet/api/system.web.configuration.httpcookiessection.requiressl) on cookies and form elements and [HttpOnly](https://docs.microsoft.com/en-us/dotnet/api/system.web.configuration.httpcookiessection.httponlycookies) on cookies in the web.config.
- Implement [customErrors](https://docs.microsoft.com/en-us/dotnet/api/system.web.configuration.customerror).
- Make sure [tracing](http://www.iis.net/configreference/system.webserver/tracing) is turned off.
- While ViewState isn't always appropriate for web development, using it can provide CSRF mitigation. To make the ViewState protect against CSRF attacks you need to set the [ViewStateUserKey](https://docs.microsoft.com/en-us/dotnet/api/system.web.ui.page.viewstateuserkey):

```csharp
protected override OnInit(EventArgs e) {
    base.OnInit(e);
    ViewStateUserKey = Session.SessionID;
}
```

If you don't use Viewstate, then look to the default main page of the ASP.NET Web Forms default template for a manual anti-CSRF token using a double-submit cookie.

```csharp
private const string AntiXsrfTokenKey = "__AntiXsrfToken";
private const string AntiXsrfUserNameKey = "__AntiXsrfUserName";
private string _antiXsrfTokenValue;
protected void Page_Init(object sender, EventArgs e)
{
    // The code below helps to protect against XSRF attacks
    var requestCookie = Request.Cookies[AntiXsrfTokenKey];
    Guid requestCookieGuidValue;
    if (requestCookie != null && Guid.TryParse(requestCookie.Value, out requestCookieGuidValue))
    {
       // Use the Anti-XSRF token from the cookie
       _antiXsrfTokenValue = requestCookie.Value;
       Page.ViewStateUserKey = _antiXsrfTokenValue;
    }
    else
    {
       // Generate a new Anti-XSRF token and save to the cookie
       _antiXsrfTokenValue = Guid.NewGuid().ToString("N");
       Page.ViewStateUserKey = _antiXsrfTokenValue;
       var responseCookie = new HttpCookie(AntiXsrfTokenKey)
       {
          HttpOnly = true,
          Value = _antiXsrfTokenValue
       };
       if (FormsAuthentication.RequireSSL && Request.IsSecureConnection)
       {
          responseCookie.Secure = true;
       }
       Response.Cookies.Set(responseCookie);
    }
    Page.PreLoad += master_Page_PreLoad;
}
protected void master_Page_PreLoad(object sender, EventArgs e)
{
    if (!IsPostBack)
    {
       // Set Anti-XSRF token
       ViewState[AntiXsrfTokenKey] = Page.ViewStateUserKey;
       ViewState[AntiXsrfUserNameKey] = Context.User.Identity.Name ?? String.Empty;
    }
    else
    {
       // Validate the Anti-XSRF token
       if ((string)ViewState[AntiXsrfTokenKey] != _antiXsrfTokenValue ||
          (string)ViewState[AntiXsrfUserNameKey] != (Context.User.Identity.Name ?? String.Empty))
       {
          throw new InvalidOperationException("Validation of Anti-XSRF token failed.");
       }
    }
}
```

- Consider [HSTS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security) in IIS. See [here](https://support.microsoft.com/en-us/help/954002/how-to-add-a-custom-http-response-header-to-a-web-site-that-is-hosted) for the procedure.
- This is a recommended `web.config` setup that handles HSTS among other things.

```xml
<?xml version="1.0" encoding="UTF-8"?>
 <configuration>
   <system.web>
     <httpRuntime enableVersionHeader="false"/>
   </system.web>
   <system.webServer>
     <security>
       <requestFiltering removeServerHeader="true" />
     </security>
     <staticContent>
       <clientCache cacheControlCustom="public"
            cacheControlMode="UseMaxAge"
            cacheControlMaxAge="1.00:00:00"
            setEtag="true" />
     </staticContent>
     <httpProtocol>
       <customHeaders>
         <add name="Content-Security-Policy"
            value="default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'" />
         <add name="X-Content-Type-Options" value="NOSNIFF" />
         <add name="X-Frame-Options" value="DENY" />
         <add name="X-Permitted-Cross-Domain-Policies" value="master-only"/>
         <add name="X-XSS-Protection" value="0"/>
         <remove name="X-Powered-By"/>
       </customHeaders>
     </httpProtocol>
     <rewrite>
       <rules>
         <rule name="Redirect to https">
           <match url="(.*)"/>
           <conditions>
             <add input="{HTTPS}" pattern="Off"/>
             <add input="{REQUEST_METHOD}" pattern="^get$|^head$" />
           </conditions>
           <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent"/>
         </rule>
       </rules>
       <outboundRules>
         <rule name="Add HSTS Header" enabled="true">
           <match serverVariable="RESPONSE_Strict_Transport_Security" pattern=".*" />
           <conditions>
             <add input="{HTTPS}" pattern="on" ignoreCase="true" />
           </conditions>
           <action type="Rewrite" value="max-age=15768000" />
         </rule>
       </outboundRules>
     </rewrite>
   </system.webServer>
 </configuration>
```

- Remove the version header by adding the following line in `Machine.config` file:

```xml
<httpRuntime enableVersionHeader="false" />
```

- Also remove the Server header using the HttpContext Class in your code.

```csharp
HttpContext.Current.Response.Headers.Remove("Server");
```

### HTTP validation and encoding

- Do not disable [validateRequest](http://www.asp.net/whitepapers/request-validation) in the `web.config` or the page setup. This value enables limited XSS protection in ASP.NET and should be left intact as it provides partial prevention of Cross Site Scripting. Complete request validation is recommended in addition to the built-in protections.
- The 4.5 version of the .NET Frameworks includes the [AntiXssEncoder](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.antixss.antixssencoder?view=netframework-4.7.2) library, which has a comprehensive input encoding library for the prevention of XSS. Use it.
- List allowable values anytime user input is accepted.
- Validate the format of URIs using [Uri.IsWellFormedUriString](https://docs.microsoft.com/en-us/dotnet/api/system.uri.iswellformeduristring).

### Forms authentication

- Use cookies for persistence when possible. `Cookieless` auth will default to [UseDeviceProfile](https://docs.microsoft.com/en-us/dotnet/api/system.web.httpcookiemode?view=netframework-4.7.2).
- Don't trust the URI of the request for persistence of the session or authorization. It can be easily faked.
- Reduce the Forms Authentication timeout from the default of *20 minutes* to the shortest period appropriate for your application. If [slidingExpiration](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.formsauthentication.slidingexpiration?view=netframework-4.7.2) is used this timeout resets after each request, so active users won't be affected.
- If HTTPS is not used, [slidingExpiration](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.formsauthentication.slidingexpiration?view=netframework-4.7.2) should be disabled. Consider disabling [slidingExpiration](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.formsauthentication.slidingexpiration?view=netframework-4.7.2) even with HTTPS.
- Always implement proper access controls.
    - Compare user provided username with `User.Identity.Name`.
    - Check roles against `User.Identity.IsInRole`.
- Use the [ASP.NET Membership provider and role provider](https://docs.microsoft.com/en-us/dotnet/framework/wcf/samples/membership-and-role-provider), but review the password storage. The default storage hashes the password with a single iteration of SHA-1 which is rather weak. The ASP.NET MVC4 template uses [ASP.NET Identity](http://www.asp.net/identity/overview/getting-started/introduction-to-aspnet-identity) instead of ASP.NET Membership, and ASP.NET Identity uses PBKDF2 by default which is better. Review the OWASP [Password Storage Cheat Sheet](Password_Storage_Cheat_Sheet.md) for more information.
- Explicitly authorize resource requests.
- Leverage role based authorization using `User.Identity.IsInRole`.

## XAML Guidance

- Work within the constraints of Internet Zone security for your application.
- Use ClickOnce deployment. For enhanced permissions, use permission elevation at runtime or trusted application deployment at install time.

## Windows Forms Guidance

- Use partial trust when possible. Partially trusted Windows applications reduce the attack surface of an application. Manage a list of what permissions your app must use, and what it may use, and then make the request for those permissions declaratively at runtime.
- Use ClickOnce deployment. For enhanced permissions, use permission elevation at runtime or trusted application deployment at install time.

## WCF Guidance

- Use HTTPS for RESTful requests regardless of HTTP method, and choose methods according to their semantics. Keep sensitive values out of URLs, which can be exposed in logs and browser history even with HTTPS. [POST can carry sensitive form data in the request body rather than the URL](https://www.rfc-editor.org/rfc/rfc9110.html#section-17.9), but POST alone does not provide confidentiality or authorization.
- Avoid [BasicHttpBinding](https://docs.microsoft.com/en-us/dotnet/api/system.servicemodel.basichttpbinding?view=netframework-4.7.2). It has no default security configuration. Use [WSHttpBinding](https://docs.microsoft.com/en-us/dotnet/api/system.servicemodel.wshttpbinding?view=netframework-4.7.2) instead.
- Use at least two security modes for your binding. Message security includes security provisions in the headers. Transport security means use of TLS. [TransportWithMessageCredential](https://docs.microsoft.com/en-us/dotnet/framework/wcf/samples/ws-transport-with-message-credential) combines the two.

## References

- [Microsoft: Secure coding guidelines for .NET](https://learn.microsoft.com/en-us/dotnet/standard/security/secure-coding-guidelines)
- [Microsoft: Introduction to Identity on ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-10.0)
