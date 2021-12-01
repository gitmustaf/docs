---
title: Authorization
description: This tutorial demonstrates how to add authorization to a Go API.
topics:
    - quickstart
    - backend
    - golang
github:
  path: 01-Authorization-RS256-BETA
contentType: tutorial
useCase: quickstart
---

<%= include('../../../_includes/_api_auth_intro') %>

<%= include('../_includes/_api_create_new') %>

<%= include('../_includes/_api_auth_preamble') %>

## Validate Access Tokens

### Download dependencies

Add a `go.mod` file to list all the dependencies to be used.

```text
// go.mod

module 01-Authorization-RS256

go 1.16

require (
	github.com/auth0/go-jwt-middleware v2.0.0-beta
	github.com/gin-contrib/cors v1.3.1
	github.com/gin-gonic/gin v1.7.4
	github.com/joho/godotenv v1.4.0
)
```

Download dependencies by running the following shell command:

```shell
go mod download
```

::: note
This example uses `gin` for routing, but you can use whichever router you want.
:::

### Configure your application

Create a `.env` file within the root of your project directory to store the app configuration, and fill in the
environment variables:

```sh
# The URL of our Auth0 Tenant Domain.
# If you're using a Custom Domain, be sure to set this to that value instead.
AUTH0_DOMAIN='${account.namespace}'

# Our Auth0 API's Identifier.
AUTH0_AUDIENCE='YOUR_API_IDENTIFIER'
```

### Create a middleware to validate Access Tokens

Access Token validation will be done in the `EnsureValidToken` middleware function which can be applied to any 
endpoints you wish to protect. If the token is valid, the resources which are served by the endpoint can be released,
otherwise a `401 Authorization` error will be returned.

Setup **go-jwt-middleware** middleware to verify Access Tokens from incoming requests.

```go
// middleware/jwt.go

package middleware

import (
	"context"
	"log"
	"net/http"
	"net/url"
	"os"
	"time"

	"github.com/auth0/go-jwt-middleware"
	"github.com/auth0/go-jwt-middleware/jwks"
	"github.com/auth0/go-jwt-middleware/validator"
	"github.com/gin-gonic/gin"
)

// CustomClaims contains custom data we want from the token.
type CustomClaims struct {
	Scope string `json:"scope"`
}

// Validate does nothing for this example, but we need
// it to satisfy josev2.CustomClaims interface.
func (c CustomClaims) Validate(ctx context.Context) error {
	return nil
}

// EnsureValidToken is a gin.HandlerFunc middleware that will check the validity of our JWT.
func EnsureValidToken() gin.HandlerFunc {
	issuerURL, err := url.Parse("https://" + os.Getenv("AUTH0_DOMAIN") + "/")
	if err != nil {
		log.Fatalf("Failed to parse the issuer url: %v", err)
	}

	provider := jwks.NewCachingProvider(issuerURL, 5*time.Minute)

	jwtValidator, err := validator.New(
		provider.KeyFunc,
		"RS256",
		issuerURL.String(),
		[]string{os.Getenv("AUTH0_AUDIENCE")},
		validator.WithCustomClaims(&CustomClaims{}),
		validator.WithAllowedClockSkew(time.Minute),
	)
	if err != nil {
		log.Fatalf("Failed to set up the jwt validator")
	}

	errorHandler := func(w http.ResponseWriter, r *http.Request, err error) {
		log.Printf("Encountered error while validating JWT: %v", err)
	}

	middleware := jwtmiddleware.New(
		jwtValidator.ValidateToken,
		jwtmiddleware.WithErrorHandler(errorHandler),
	)

	return func(ctx *gin.Context) {
		var encounteredError = true
		var handler http.HandlerFunc = func(w http.ResponseWriter, r *http.Request) {
			encounteredError = false
			ctx.Request = r
			ctx.Next()
		}

		middleware.CheckJWT(handler).ServeHTTP(ctx.Writer, ctx.Request)

		if encounteredError {
			ctx.AbortWithStatusJSON(
				http.StatusUnauthorized,
				map[string]string{"message": "Failed to validate JWT."},
			)
		}
	}
}
```

<%= include('../_includes/_api_jwks_description') %>


## Protect API Endpoints

To protect individual routes, pass `middleware` (defined above) to the `gin` route.

```go
// main.go

package main

import (
	"log"
	"net/http"

	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
	"github.com/joho/godotenv"

	"01-Authorization-RS256/middleware"
)

func main() {
	if err := godotenv.Load(); err != nil {
		log.Fatalf("Error loading the .env file: %v", err)
	}

	router := gin.Default()

	router.Use(cors.New(
		cors.Config{
			AllowOrigins:     []string{"http://localhost:3000"},
			AllowCredentials: true,
			AllowHeaders:     []string{"Authorization"},
		},
	))

	// This route is always accessible.
	router.Any("/api/public", func(ctx *gin.Context) {
		response := map[string]string{
			"message": "Hello from a public endpoint! You don't need to be authenticated to see this.",
		}
		ctx.JSON(http.StatusOK, response)
	})

	// This route is only accessible if the user has a valid access_token.
	router.GET(
		"/api/private",
		middleware.EnsureValidToken(),
		func(ctx *gin.Context) {
			response := map[string]string{
				"message": "Hello from a private endpoint! You need to be authenticated to see this.",
			}
			ctx.JSON(http.StatusOK, response)
		},
	)

	log.Print("Server listening on http://localhost:3010")
	if err := http.ListenAndServe("0.0.0.0:3010", router); err != nil {
		log.Fatalf("There was an error with the http server: %v", err)
	}
}
```

### Validate scopes

The `middleware` above verifies that the Access Token included in the request is valid; however, it doesn't yet include
any mechanism for checking that the token has the sufficient **scope** to access the requested resources.

Create a function to check and ensure the Access Token has the correct scope before returning a successful response.

```go
// 👆 We're continuing from the steps above. Append this to your middleware/jwt.go file.

// HasScope checks whether our claims have a specific scope.
func (c CustomClaims) HasScope(expectedScope string) bool {
    result := strings.Split(c.Scope, " ")
    for i := range result {
        if result[i] == expectedScope {
            return true
        }
    }

    return false
}
```

Use this function in the endpoint that requires the scope `read:messages`.

```go
// 👆 We're continuing from the steps above. Append this to your main.go file.

func main() {
    // ...
    
    // This route is only accessible if the user has a
    // valid access_token with the read:messages scope.
    router.GET(
        "/api/private-scoped",
        middleware.EnsureValidToken(),
        func(ctx *gin.Context) {
            token := ctx.Request.Context().Value(jwtmiddleware.ContextKey{}).(*josev2.UserContext)
            
			claims := token.CustomClaims.(*middleware.CustomClaims)
            if !claims.HasScope("read:messages") {
                response := map[string]string{"message": "Insufficient scope."}
                ctx.JSON(http.StatusForbidden, response)
                return
            }
            
            response := map[string]string{
                "message": "Hello from a private endpoint! You need to be authenticated to see this.",
            }
            ctx.JSON(http.StatusOK, response)
        },
    )
    
    // ...
}
```

In this example, only the `read:messages` scope is checked. You may want to extend the `HasScope` function or make it
a standalone middleware that accepts multiple scopes to fit your use case.
