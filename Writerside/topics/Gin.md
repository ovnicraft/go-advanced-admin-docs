# Gin

## Introduction

Welcome to the comprehensive guide on integrating the **Go Advanced Admin Panel** with the **Gin** web framework. This 
guide aims to provide you with detailed instructions on how to use and customize the Gin integration to seamlessly 
incorporate the admin panel into your Gin applications.

[Gin](https://gin-gonic.com/) is a high-performance HTTP web framework that can be used to build web applications and microservices in Go. By integrating the admin panel with Gin, you can leverage Gin's 
capabilities while providing a powerful administrative interface for managing your application's data and 
configurations.

> **Note:** This guide assumes that you have already completed the [Gin Quick Start Guide](Gin-Quick-Start.md) and 
> are familiar with setting up a basic admin panel with Gin. If you haven't, please refer to the Quick Start Guide 
> before proceeding.

## Prerequisites

Before diving into the integration, ensure you have the following:

- **Go** installed on your system.
- A working **Gin** application.
- Familiarity with **GORM** (Go ORM) for database interactions.
- The following packages installed:

  ```bash
  go get github.com/go-advanced-admin/admin
  go get github.com/go-advanced-admin/web-gin
  go get github.com/go-advanced-admin/orm-gorm
  ```

## Setting Up the Gin Integration

### Import Necessary Packages

In your main application file, import the required packages:

```go
import (
    "log"

    "github.com/go-advanced-admin/admin"
    "github.com/go-advanced-admin/web-gin"
    "github.com/go-advanced-admin/orm-gorm"
    "github.com/gin-gonic/gin"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)
```

### Initialize Gin and the Admin Panel

Set up your Gin instance and initialize the admin panel with GORM and Gin integrations:

```go
func main() {
    // Initialize Gin
    r := gin.Default()

    // Initialize GORM with a SQLite database (for example)
    db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }

    // Create a new GORM integrator
    gormIntegrator := admingorm.NewIntegrator(db)

    // Create a new Gin integrator
    ginIntegrator := admingin.NewIntegrator(r.Group(""))

    // Define your permission checker function
    permissionChecker := func(
        request admin.PermissionRequest, ctx interface{},
    ) (bool, error) {
        // Implement your permission logic here
        return true, nil
    }

    // Create a new admin panel
    adminPanel, err := admin.NewPanel(
        gormIntegrator, ginIntegrator, permissionChecker, nil, 
    )
    if err != nil {
        log.Fatal(err)
    }

    // Register applications and models (explained later)

    // Start the Gin server
    r.Run(":8080")
}
```

## Using the Gin Integration

### Handling Routes

The Gin integrator handles route registration by mapping admin panel routes to Gin handlers. When you use the 
`HandleRoute` method, routes are added to the Gin group:

```go
func (i *Integrator) HandleRoute(method, path string, handler admin.HandlerFunc) {
    i.group.Handle(method, path, func(c *gin.Context) {
        codes := []int{http.StatusFound, http.StatusMovedPermanently, http.StatusSeeOther}
        code, body := handler(c)
        if slices.Contains(codes, int(code)) {
            c.Redirect(int(code), body)
        }

        c.Data(int(code), "text/html; charset=utf-8", []byte(body))
    })
}
```

This method ensures that HTTP responses from the admin panel are correctly handled within the Gin framework.

### Serving Static Assets

To serve static assets (like CSS and JavaScript files) required by the admin panel, use the `ServeAssets` method:

```go
func (i *Integrator) ServerAssets(prefix string, renderer admin.TemplateRenderer) {
    i.group.GET(fmt.Sprintf("%s/*filepath", prefix), func(c *gin.Context) {
        fileName := c.Param("filepath")
        fileData, err := renderer.GetAsset(fileName)
        if err != nil {
            c.String(http.StatusNotFound, "File not found")
            return
        }
        contentType := mime.TypeByExtension(filepath.Ext(fileName))

        if contentType == "" {
            contentType = "application/octet-stream"
        }

        c.Data(http.StatusOK, contentType, fileData)
    })
}
```

This method uses the `admin.TemplateRenderer` to retrieve asset files and serves them through Gin's routing system.

### Extracting Request Data

The Gin integrator provides methods to extract query parameters, path parameters, request methods, and form data from 
the Gin context:

```go
func (i *Integrator) GetQuery(ctx interface{}, name string) string {
    c, ok := ctx.(*gin.Context)
    if !ok {
        return ""
    }
    return c.Query(name)
}

func (i *Integrator) GetPathParam(ctx interface{}, name string) string {
    c, ok := ctx.(*gin.Context)
    if !ok {
        return ""
    }
    return c.Param(name)
}

func (i *Integrator) GetRequestMethod(ctx interface{}) string {
    c, ok := ctx.(*gin.Context)
    if !ok {
        return ""
    }
    return c.Request.Method
}

func (i *Integrator) GetFormData(ctx interface{}) map[string][]string {
    c, ok := ctx.(*gin.Context)
    if !ok {
        return nil
    }
    if err := c.Request.ParseForm(); err != nil {
        return nil
    }
    return c.Request.Form
}
```

These methods are used by the admin panel to handle HTTP requests and extract necessary data for processing.

## Customizing the Gin Integration

### Custom Route Handling

You can customize route handling by adding middleware or modifying the Gin group before integrating it with the admin 
panel:

```go
// Create a new Gin group with middleware
adminGroup := r.Group("/admin", yourMiddleware)

// Create the Gin integrator with the customized group
ginIntegrator := admingin.NewIntegrator(adminGroup)

// Initialize the admin panel with the custom integrator
adminPanel, err := admin.NewPanel(
    gormIntegrator, ginIntegrator, permissionChecker, nil, 
)
if err != nil {
    log.Fatal(err)
}
```

### Error Handling

Gin allows you to define custom error handlers. You can set a global error handler to manage errors occurring within 
the admin panel routes:

```go
r.Use(gin.CustomRecovery(func(c *gin.Context, recovered interface{}) {
    if err, ok := recovered.(string); ok {
        c.String(http.StatusInternalServerError, fmt.Sprintf("error: %s", err))
    }
    c.AbortWithStatus(http.StatusInternalServerError)
}))
```

### Advanced Customization

For advanced customization, you can
[implement your own version of the `admin.WebIntegrator`](Building-a-Custom-Web-Framework-Integration.md) interface to 
have full control over request handling:

```go
type CustomIntegrator struct {
    group *gin.RouterGroup
}

// Implement the required methods
func (i *CustomIntegrator) HandleRoute(
    method, path string, handler admin.HandlerFunc, 
) {
    // Custom route handling
}

func (i *CustomIntegrator) ServeAssets(
    prefix string, renderer admin.TemplateRenderer, 
    ) {
    // Custom asset serving
}

// ... Implement other methods
```

By creating a custom integrator, you can integrate the admin panel into your Gin application in a way that best suits 
your requirements.

## Understanding the GORM Integration

The admin panel uses an ORM integrator to interact with the database. The GORM integration (`admingorm`) allows you to 
manage your database models seamlessly.

### Registering Models with GORM

Define your models using GORM and register them with the admin panel:

```go
type User struct {
    ID       uint   `gorm:"primaryKey"`
    Name     string
    Email    string
    Password string
}

func main() {
    // ... Initialization code

    // Auto-migrate your models
    db.AutoMigrate(&User{})

    // Register an application
    userApp, err := adminPanel.RegisterApp(
        "user_app", "User Management", nil, 
    )
    if err != nil {
        log.Fatal(err)
    }

    // Register the User model
    _, err = userApp.RegisterModel(&User{}, nil)
    if err != nil {
        log.Fatal(err)
    }

    // Start the Gin server
    r.Run(":8080")
}
```

## Contributing to the Gin Integration

We welcome contributions to enhance the Gin integration. If you encounter issues or have ideas for improvements, 
please:

- **Report Issues:** Use the [GitHub Issues](https://github.com/go-advanced-admin/admingin/issues) page to report bugs 
or request features.
- **Submit Pull Requests:** If you have code changes, submit a pull request following our
[Contribution Guidelines](Contributing.md).
- **Join Discussions:** Participate in discussions to share your insights and collaborate with the community.

Your contributions help make the project better for everyone!

## Conclusion

Integrating the Go Advanced Admin Panel with Gin allows you to build a powerful and customizable administrative 
interface for your web applications. By understanding and utilizing the Gin and GORM integrations, you can manage your 
application's data effectively and tailor the admin panel to meet your specific needs.

Explore other sections of the documentation to learn more about advanced features, customization options, and best 
practices.

---

> For further assistance or questions, please refer to the [official documentation](Welcome.md) or
> [reach out to the community](https://github.com/go-advanced-admin/web-gin/discussions).*

> This section of the documentation was written with the help of [Infuzu AI](https://infuzu.com)'s tools.
{style="note"}