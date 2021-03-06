<%= title("Resources") %>

<%= vimeo("212302823") %>

```bash
$ buffalo g resource --help

Generates a new actions/resource file
Usage:
  buffalo generate resource [name] [flags]

Aliases:
  resource, r
```

```bash
$ buffalo g resource users name email bio:nulls.Text

--> actions/users.go
--> actions/users_test.go
--> locales/users.en-us.yaml
--> templates/users/_form.html
--> templates/users/edit.html
--> templates/users/index.html
--> templates/users/new.html
--> templates/users/show.html
--> buffalo db g model user name email bio:nulls.Text
--> models/user.go
--> models/user_test.go
--> goimports -w .
> migrations/20170410180211_create_users.up.fizz
> migrations/20170410180211_create_users.down.fizz
--> goimports -w .
```

```go
// actions/app.go
package actions

import (
  "github.com/gobuffalo/buffalo"
  "github.com/gobuffalo/buffalo/middleware"
  "github.com/gobuffalo/buffalo/middleware/csrf"
  "github.com/gobuffalo/buffalo/middleware/i18n"

  "github.com/markbates/coke/models"

  "github.com/gobuffalo/envy"
  "github.com/gobuffalo/packr"
)

// ENV is used to help switch settings based on where the
// application is being run. Default is "development".
var ENV = envy.Get("GO_ENV", "development")
var app *buffalo.App
var T *i18n.Translator

// App is where all routes and middleware for buffalo
// should be defined. This is the nerve center of your
// application.
func App() *buffalo.App {
  if app == nil {
    app = buffalo.Automatic(buffalo.Options{
      Env:         ENV,
      SessionName: "_coke_session",
    })

    if ENV == "development" {
      app.Use(middleware.ParameterLogger)
    }
    if ENV != "test" {
      // Protect against CSRF attacks. https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)
      // Remove to disable this.
      app.Use(csrf.Middleware)
    }

    // Wraps each request in a transaction.
    //  c.Value("tx").(*pop.PopTransaction)
    // Remove to disable this.
    app.Use(middleware.PopTransaction(models.DB))

    // Setup and use translations:
    var err error
    if T, err = i18n.New(packr.NewBox("../locales"), "en-US"); err != nil {
      app.Stop(err)
    }
    app.Use(T.Middleware())

    app.GET("/", HomeHandler)

    app.ServeFiles("/assets", packr.NewBox("../public/assets"))
    app.Resource("/users", UsersResource{&buffalo.BaseResource{}})
  }

  return app
}
```

```go
// actions/users.go
package actions

import (
  "github.com/gobuffalo/buffalo"
  "github.com/markbates/coke/models"
  "github.com/markbates/pop"
  "github.com/pkg/errors"
)

// This file is generated by Buffalo. It offers a basic structure for
// adding, editing and deleting a page. If your model is more
// complex or you need more than the basic implementation you need to
// edit this file.

// Following naming logic is implemented in Buffalo:
// Model: Singular (User)
// DB Table: Plural (Users)
// Resource: Plural (Users)
// Path: Plural (/users)
// View Template Folder: Plural (/templates/users/)

// UsersResource is the resource for the user model
type UsersResource struct {
  buffalo.Resource
}

// List gets all Users. This function is mapped to the path
// GET /users
func (v UsersResource) List(c buffalo.Context) error {
  // Get the DB connection from the context
  tx := c.Value("tx").(*pop.Connection)
  users := &models.Users{}
  // You can order your list here. Just change
  err := tx.All(users)
  // to:
  // err := tx.Order("create_at desc").All(users)
  if err != nil {
    return errors.WithStack(err)
  }
  // Make users available inside the html template
  c.Set("users", users)
  return c.Render(200, r.HTML("users/index.html"))
}

// Show gets the data for one User. This function is mapped to
// the path GET /users/{user_id}
func (v UsersResource) Show(c buffalo.Context) error {
  // Get the DB connection from the context
  tx := c.Value("tx").(*pop.Connection)
  // Allocate an empty User
  user := &models.User{}
  // To find the User the parameter user_id is used.
  err := tx.Find(user, c.Param("user_id"))
  if err != nil {
    return errors.WithStack(err)
  }
  // Make user available inside the html template
  c.Set("user", user)
  return c.Render(200, r.HTML("users/show.html"))
}

// New renders the form for creating a new user.
// This function is mapped to the path GET /users/new
func (v UsersResource) New(c buffalo.Context) error {
  // Make user available inside the html template
  c.Set("user", &models.User{})
  return c.Render(200, r.HTML("users/new.html"))
}

// Create adds a user to the DB. This function is mapped to the
// path POST /users
func (v UsersResource) Create(c buffalo.Context) error {
  // Allocate an empty User
  user := &models.User{}
  // Bind user to the html form elements
  err := c.Bind(user)
  if err != nil {
    return errors.WithStack(err)
  }
  // Get the DB connection from the context
  tx := c.Value("tx").(*pop.Connection)
  // Validate the data from the html form
  verrs, err := tx.ValidateAndCreate(user)
  if err != nil {
    return errors.WithStack(err)
  }
  if verrs.HasAny() {
    // Make user available inside the html template
    c.Set("user", user)
    // Make the errors available inside the html template
    c.Set("errors", verrs)
    // Render again the new.html template that the user can
    // correct the input.
    return c.Render(422, r.HTML("users/new.html"))
  }
  // If there are no errors set a success message
  c.Flash().Add("success", "User was created successfully")
  // and redirect to the users index page
  return c.Redirect(302, "/users/%s", user.ID)
}

// Edit renders a edit formular for a user. This function is
// mapped to the path GET /users/{user_id}/edit
func (v UsersResource) Edit(c buffalo.Context) error {
  // Get the DB connection from the context
  tx := c.Value("tx").(*pop.Connection)
  // Allocate an empty User
  user := &models.User{}
  err := tx.Find(user, c.Param("user_id"))
  if err != nil {
    return errors.WithStack(err)
  }
  // Make user available inside the html template
  c.Set("user", user)
  return c.Render(200, r.HTML("users/edit.html"))
}

// Update changes a user in the DB. This function is mapped to
// the path PUT /users/{user_id}
func (v UsersResource) Update(c buffalo.Context) error {
  // Get the DB connection from the context
  tx := c.Value("tx").(*pop.Connection)
  // Allocate an empty User
  user := &models.User{}
  err := tx.Find(user, c.Param("user_id"))
  if err != nil {
    return errors.WithStack(err)
  }
  // Bind user to the html form elements
  err = c.Bind(user)
  if err != nil {
    return errors.WithStack(err)
  }
  verrs, err := tx.ValidateAndUpdate(user)
  if err != nil {
    return errors.WithStack(err)
  }
  if verrs.HasAny() {
    // Make user available inside the html template
    c.Set("user", user)
    // Make the errors available inside the html template
    c.Set("errors", verrs)
    // Render again the edit.html template that the user can
    // correct the input.
    return c.Render(422, r.HTML("users/edit.html"))
  }
  // If there are no errors set a success message
  c.Flash().Add("success", "User was updated successfully")
  // and redirect to the users index page
  return c.Redirect(302, "/users/%s", user.ID)
}

// Destroy deletes a user from the DB. This function is mapped
// to the path DELETE /users/{user_id}
func (v UsersResource) Destroy(c buffalo.Context) error {
  // Get the DB connection from the context
  tx := c.Value("tx").(*pop.Connection)
  // Allocate an empty User
  user := &models.User{}
  // To find the User the parameter user_id is used.
  err := tx.Find(user, c.Param("user_id"))
  if err != nil {
    return errors.WithStack(err)
  }
  err = tx.Destroy(user)
  if err != nil {
    return errors.WithStack(err)
  }
  // If there are no errors set a flash message
  c.Flash().Add("success", "User was destroyed successfully")
  // Redirect to the users index page
  return c.Redirect(302, "/users")
}
```

```go
// models/users.go
package models

import (
  "encoding/json"
  "time"

  "github.com/markbates/pop"
  "github.com/markbates/pop/nulls"
  "github.com/markbates/validate"
  "github.com/markbates/validate/validators"
  "github.com/satori/go.uuid"
)

type User struct {
  ID        uuid.UUID    `json:"id" db:"id"`
  CreatedAt time.Time    `json:"created_at" db:"created_at"`
  UpdatedAt time.Time    `json:"updated_at" db:"updated_at"`
  Name      string       `json:"name" db:"name"`
  Email     string       `json:"email" db:"email"`
  Bio       nulls.String `json:"bio" db:"bio"`
}

// String is not required by pop and may be deleted
func (u User) String() string {
  ju, _ := json.Marshal(u)
  return string(ju)
}

// Users is not required by pop and may be deleted
type Users []User

// String is not required by pop and may be deleted
func (u Users) String() string {
  ju, _ := json.Marshal(u)
  return string(ju)
}

// Validate gets run everytime you call a "pop.Validate" method.
// This method is not required and may be deleted.
func (u *User) Validate(tx *pop.Connection) (*validate.Errors, error) {
  return validate.Validate(
    &validators.StringIsPresent{Field: u.Name, Name: "Name"},
    &validators.StringIsPresent{Field: u.Email, Name: "Email"},
  ), nil
}

// ValidateSave gets run everytime you call "pop.ValidateSave" method.
// This method is not required and may be deleted.
func (u *User) ValidateSave(tx *pop.Connection) (*validate.Errors, error) {
  return validate.NewErrors(), nil
}

// ValidateUpdate gets run everytime you call "pop.ValidateUpdate" method.
// This method is not required and may be deleted.
func (u *User) ValidateUpdate(tx *pop.Connection) (*validate.Errors, error) {
  return validate.NewErrors(), nil
}
```

```fizz
// migration
create_table("users", func(t) {
  t.Column("id", "uuid", {"primary": true})
  t.Column("name", "string", {})
  t.Column("email", "string", {})
  t.Column("bio", "text", {"null": true})
})
```

```html
// users/index.html
&lt;h1&gt;Users&lt;/h1&gt;

&lt;p&gt;
  &lt;a href="&lt;%= newUserPath() %&gt;" class="btn btn-primary"&gt;Create New User&lt;/a&gt;
&lt;/p&gt;

&lt;table class="table table-striped"&gt;
  &lt;thead&gt;
  &lt;th&gt;Name&lt;/th&gt;
  &lt;th&gt;Email&lt;/th&gt;
  &lt;th&gt;&amp;amp;nbsp;&lt;/th&gt;
  &lt;/thead&gt;
  &lt;tbody&gt;
    &lt;%= for (user) in users { %&gt;
      &lt;tr&gt;
        &lt;td&gt;&lt;%= user.Name %&gt;&lt;/td&gt;
        &lt;td&gt;&lt;%= user.Email %&gt;&lt;/td&gt;
        &lt;td&gt;
          &lt;div class="pull-right"&gt;
            &lt;a href="&lt;%= userPath({ user_id: user.ID }) %&gt;" class="btn btn-info"&gt;View&lt;/a&gt;
            &lt;a href="&lt;%= editUserPath({ user_id: user.ID }) %&gt;" class="btn btn-warning"&gt;Edit&lt;/a&gt;
            &lt;a href="&lt;%= userPath({ user_id: user.ID }) %&gt;" data-method="DELETE" data-confirm="Are you sure?" class="btn btn-danger"&gt;Destroy&lt;/a&gt;
          &lt;/div&gt;
        &lt;/td&gt;
      &lt;/tr&gt;
    &lt;% } %&gt;
  &lt;/tbody&gt;
&lt;/table&gt;
```

```html
// users/show.html
&lt;h1&gt;Users#Show&lt;/h1&gt;

&lt;ul class="list-unstyled list-inline"&gt;
  &lt;li&gt;&lt;a href="&lt;%= usersPath() %&gt;" class="btn btn-info"&gt;Back to all Users&lt;/a&gt;&lt;/li&gt;
  &lt;li&gt;&lt;a href="&lt;%= editUserPath({ user_id: user.ID })%&gt;" class="btn btn-warning"&gt;Edit&lt;/a&gt;&lt;/li&gt;
  &lt;li&gt;&lt;a href="&lt;%= userPath({ user_id: user.ID })%&gt;" data-method="DELETE" data-confirm="Are you sure?" class="btn btn-danger"&gt;Destroy&lt;/a&gt;
&lt;/ul&gt;

&lt;p&gt;
  &lt;strong&gt;Name&lt;/strong&gt;: &lt;%= user.Name %&gt;
&lt;/p&gt;
&lt;p&gt;
  &lt;strong&gt;Email&lt;/strong&gt;: &lt;%= user.Email %&gt;
&lt;/p&gt;
&lt;p&gt;
  &lt;strong&gt;Bio&lt;/strong&gt;: &lt;%= user.Bio %&gt;
&lt;/p&gt;
```

```html
// users/new.html
&lt;h1&gt;New User&lt;/h1&gt;

&lt;%= form_for(user, {action: usersPath(), method: "POST"}) { %&gt;
  &lt;%= partial("users/form.html") %&gt;
  &lt;a href="&lt;%= usersPath() %&gt;" class="btn btn-warning" data-confirm="Are you sure?"&gt;Cancel&lt;/a&gt;
&lt;% } %&gt;
```

```html
// users/_form.html
&lt;%= f.InputTag("Name") %&gt;
&lt;%= f.InputTag("Email") %&gt;
&lt;%= f.TextArea("Bio", {rows: 10}) %&gt;
&lt;button class="btn btn-success" role="submit"&gt;Save&lt;/button&gt;
```

```html
// users/edit.html
&lt;h1&gt;Edit User&lt;/h1&gt;

&lt;%= form_for(user, {action: userPath({ user_id: user.ID }), method: "PUT"}) { %&gt;
  &lt;%= partial("users/form.html") %&gt;
  &lt;a href="\&lt;%= userPath({ user_id: user.ID }) %&gt;" class="btn btn-warning" data-confirm="Are you sure?"&gt;Cancel&lt;/a&gt;
&lt;% } %&gt;
```

You can remove files generated by this generator by running:

```bash
$ buffalo destroy resource users
```

This command will ask you which files you want to remove, you can either answer each of the questions with y/n or you can pass the -y flag to the command like:

```bash
$ buffalo destroy resource users -y
```

Or in short form:

```bash
$ buffalo d r users -y
```
