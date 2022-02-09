# Access Control

## Configuration of the Principal

The Access Control sublanguage is supported by a session entity that holds information 
regarding the currently logged in user. This session entity is called the securityContext 
and is configured as follows:
  
    principal is User with credentials name, password
  
This states that the User entity will be the entity representing a logged in user.
The credentials are not used in the current implementation (the idea is to derive a default 
login template). The resulting generated session entity will be:

    session securityContext
    {
      principal -> User
      loggedIn :: Bool := this.principal != null
    }

Note that this principal declaration is used to enable access control in the application.

It will also generate `authentication()` (both login and logout), `login()`, and `logout()` templates, and an `authenticate` function that takes the credentials as arguments and, if they are correct, returns true and sets the principal (only String/Email/Secret-type credential properties are allowed).

## Authentication

Authentication can be added manually, instead of using the generated authentication templates. Here is a small example application with custom authentication:

     principal is User with credentials name, password
  
    entity User{
      name :: String
      password :: Secret
    }
  
    define login(){
      var username := ""
      var password : Secret := ""
      form{ 
        label("Name: "){ input(username) }
        label("Password: "){ input(password) }
        captcha()
        submit login() {"Log In"}
      }
      action login(){
        validate(authenticate(username,password), 
          "The login credentials are not valid.");
        message("You are now logged in.");
      }
    }
  
    define logout(){
      "Logged in as " output(securityContext.principal)
      form{
        submitlink logout() {"Log Out"}
      }
      action logout(){
        securityContext.principal := null;
      }
    }
      
    define page root(){
      login()
      " "
      logout()
    }
    
    init{
      var u1 : User := 
        User{ name := "test" password := ("test" as Secret).digest() };
      u1.save();
    }
    
    access control rules
      
      rule page root(){
        true
      }

When storing a secret property you need to create a digest of it:
<verbatim>
  newUser.password := newUser.password.digest();
</verbatim>
This makes sure the secret property is stored encrypted.
A digest can be compared with an entered string using the check method:
<verbatim>
  us.password.check(enteredpassword)
</verbatim>

## Protecting Resources

The default policy is to deny access to all pages and ajax templates, the rules determine what the conditions for allowing access are. Regular templates are accessible by default, however, you can add additional access control rules on templates to limit their accessibility.

A simple rule protecting the editUser page to be only accessable by the user
being edited looks like this:
<verbatim>
access control rules

  rule page editUser(u:User){
    u == principal     
  }
</verbatim>
An analysis of this rule:

* access control rules: a rules section is started with this declaration, multiple rules can follow. To go back to a normal section, use `section some description`.
* rule: A keyword for Access Control rules
* page: The type of resource being protected here, all the types available
  in the Access Control DSL for WebDSL are: page, action, template, function. The rules 
  on pages protect the viewing of pages, action rules protect the execution of actions, 
  template rules determine whether a template is visible in the including page, and finally 
  rules on functions are lifted to the action invoking the function.
* editUser: The name of the resource the rule will apply to.
* (u:User): The arguments (if any) of the resource, the types of the
  arguments are used when matching. This also specifies what variables can be
  used in the checks.
* u = principal: the check that determines whether access to
  this resource is allowed, this check is typechecked to be a correct boolean
  expression. The use of principal implies that the
  securityContext is not null and the user is logged in (these extra checks are
  generated automatically).

Matching can be done a bit more freely using a trailing * as 
wildcard character, both in resource name and arguments:
<verbatim>
rule page viewUs*(*){
  true
}
</verbatim>

When more fine-grained control is needed for rules, it is possible to specify 
nested rules. This implies that the nested rule is only valid for usage of that 
resource inside the parent resource. The allowed combinations are page - action,
 template - action, page - template. The next example shows nested rules for actions in a page:
<verbatim>
  rule page editDocument(d:Document){
    d.author == principal
    rule action save(){
      d.author == principal
    }
    rule action cancel(){
      d.author == principal
    }      
  }
</verbatim>

This flexibility is often not necessary, and it is also inconvenient
having to explicitly allow all the actions on the page, for these reasons some
extra desugaring rules were added. When specifying a check on a page or template without
nested checks, a generic action rules block with the same check is added to it by
default. For example:
<verbatim>
  rule page editDocument(d:Document){
    d.author == principal     
  }
</verbatim>
becomes
<verbatim>
  rule page editDocument(d:Document){
    d.author == principal
    rule action *(*)
    {
      d.author == principal
    }    
  }
</verbatim>

## Reuse in Access Control rules

Predicates are functions consisting of one boolean 
expression, which allows reusing complicated expressions, or simply giving
better structure to the policy implementation. An example of a predicate:
<verbatim>
  predicate mayViewDocument (u:User, d:Document){
    d.author == principal
    || u in d.allowedUsers
  }
  rule page viewDocument(d:Document){
    mayViewDocument(principal,d)
  }
  rule page showDocument(d:Document){
    mayViewDocument(principal,d)
  }
</verbatim>

Pointcuts are groups of resources to which conditions can be specified at once.
Especially the open parts of the web application are easy to handle with
pointcuts, an example:
<verbatim>
  pointcut openSections(){
    page home(),
    page createDocument(),
    page createUser(),
    page viewUser(*)
  }
  rule pointcut openSections(){
    true
  }
</verbatim>
Pointcuts can also be used with parameters:
<verbatim>
  pointcut ownUserSections(u:User){
    page showUser(u),
    page viewUser(u),
    page someUserTask(u,*)
  }
  rule pointcut ownUserSections(u:User){
    u == principal
  }
</verbatim>
Note that each parameter must be used in each pointcut element, this indicates the value to be used as argument for the pointcut check. A wildcard * can follow to indicate that there may be additional arguments.

## Inferring Visibility

A disabled page or action redirects to a very simple page stating access denied.
Since this is not very user friendly, the visibility of navigate links and action 
buttons/links are automatically made conditional using the same check as the corresponding
resource. An example conditional navigate:
<verbatim>
if(mayViewDocument(securityContext.principal,d)){
  navigate(viewDocument(d)){ "view " output(d.title) } 
}
</verbatim>

When using conditional forms it is often more convenient to put the form in a template, 
and control the visibility by a rule on the template.

## Using Entities

Access Control policies that rely on extra data can create new or extend existing properties. 
An example of extending an entity is adding a set of users property to a document representing 
the users allowed access to that document:
<verbatim>
  extend entity Document{
    allowedUsers -> Set<User>
  }
</verbatim>

## Administration of Access Control

Administration of Access Control in WebDSL is done by the normal WebDSL page definitions.
All the data of the Access Control policy is integrated into the WebDSL
application. An option is to incorporate the administration into an existing page with a
template. This example illustrates the use of a template for administration:
<verbatim>
  define allowedUsersRow(document:Document){
    row{ "Allowed Users:" input(document.allowedUsers) }
  }
</verbatim>
The template call for this template is added to the editDocument page:
<verbatim>
  table{
    row{ "Title:" input(document.title) }
    row{ "Text:" input(document.text) }
    row{ "Author:" input(document.author) }  
    allowedUsersRow(document)
  }
</verbatim>
By using a template the Access Control can be disabled easily by not 
including the access control definitions and the template. The unresolved template definitions will give a warning but the page will generate normally and ignore the template call.

## Minimal Access Control Example

	application minimalac
	
	  entity User {
	  	name :: String
	  	password :: Secret
	  }
	  
	  init{
	  	var u := User{ name := "1" password := ("1" as Secret).digest()  };
	  	u.save();
	  }
	  
	  define page root(){
	  	authentication()
	  	" "
	  	navigate protectedPage() { "go" }
	  }
	  
	  define page protectedPage(){ "access granted" }
	  
	  principal is User with credentials name, password
	  
	  access control rules
	  
	    rule page root(){true}
	    rule page protectedPage(){loggedIn()}
