---
layout: post
title:  "Making up a Terraform provider (plugin) for SQLite: v0.1.0"
date:   2021-09-01 13:28:29 -0700
categories: posts
tags: [Terraform, Databases, DevOps, SRE, Infrastructure management, Go, Golang]
author: Konstantin Vasilev
---

This post is mostly a diary of mine about how I dig up on creating a Terraform provider.

So many posts these days you can find on the Internet about Terraform and how to create a custom provider (plugin) for it. But most of these articles simply rephrase official documentation and tutorials, leading you through the creation process of an abstract REST API.

Still Terraform is claimed to be a tool to manage anything, so I decided to make up a provider for anything else than just another REST API. I decided to implement a management layer for...SQLite database engine. :)

The complete code for version [0.1.0](https://github.com/Burmuley/terraform-provider-sqlite/tree/v0.1.0) can be found in [`my github`](https://github.com/Burmuley/terraform-provider-sqlite/tree/v0.1.0).

So, let's start the journey!

## Why SQLite?

Yeah, it's obviously the latter tool/service anyone would need a Terraform to manage. But the reason I've chosen this one is the lack of any services or extra tools you need to install to start using it. Eventually SQLite is just a library you need to import while writing code in your preferred programming language and this is it! Still it has some static resources to be managed by a [`IaaC`](https://en.wikipedia.org/wiki/Infrastructure_as_code) tool like Terraform.

For the first version of the provider I considered the following capabilities:
1. Create a database itself (it's just a file mkay)
2. Create and delete tables in this database
3. Create and delete indexes, at least the simplest cases

Any SQLite driver creates a database when you `open` it (in case it does not exist of course).
There is also an option to `attach` multiple databases to a single connection, but this case is out of the scope of this article.

Creating tables in SQLite is very simple with the following statement:
```sql
CREATE TABLE table_name (
  column_1 data_type PRIMARY KEY,
  column_2 data_type NOT NULL,
  column_3 data_type DEFAULT 0,
);
```

And for indexes the statement is even more simple:
```sql
CREATE INDEX index_name ON table_name(column_1, column_2, ...);
```

Here we go, I just need to generate such SQL statement from the input configuration defined in HCL.
Since I'm going to make the plugin with `Go` our best option is `text/template` package. I'll get back to this later in this post.

I selected [`modernc`](modernc.org/sqlite) SQLite driver implementation as it's in pure Go what reduces headache when building and debugging a plugin.

The only downside of this driver it's not thread-safe (remember we work with just a file). So I made up a tiny wrapper involving [`sync.Mutex`](https://pkg.go.dev/sync#Mutex) to `Lock` and `Unlock` database connection. This wrapper mirrors some of [`database/sql.DB`](https://pkg.go.dev/database/sql#DB) methods like `Open`, `Query`, `Exec` and represents a simplest data structure:
```golang
type sqLiteWrapper struct {
  *sync.Mutex
  db     *sql.DB
  dbPath string
}
```

The complete source code you can find in [`sqlite/sqlite.go`](https://github.com/Burmuley/terraform-provider-sqlite/blob/v0.1.0/sqlite/sqlite.go) file.

Quick example for `Query` method implementation for this wrapper:
```golang
func (s *sqLiteWrapper) Query(query string, args ...interface{}) (*sql.Rows, error) {
  if s.db == nil {
    return nil, errors.New("database not initialized")
  }

  s.Lock()
  defer s.Unlock()
  return s.db.Query(query, args...)
}
```
<br>
At this point it's all clear about SQLite side, let's shed a light on some Terraform internals.

## Terraform plugin system

Before I started digging this topic, I was sure Terraform uses the standard [`Go plugin`](https://pkg.go.dev/plugin) mechanism for providers implementation. But eventually I realized they implemented their own approach and it's based on [`RPC`](https://en.wikipedia.org/wiki/Remote_procedure_call) to support modules independence from Terraform Core (for details on [`Go plugin`](https://pkg.go.dev/plugin) limitations see [`this nice article`](https://eli.thegreenplace.net/2021/plugins-in-go/)).

So at the end of the day HashiCorp's [`plugin`](https://github.com/hashicorp/go-plugin) (provider) is a standalone binary that is launched by Terraform Core when you run `terraform plan/apply` and starts an [`RPC`](https://en.wikipedia.org/wiki/Remote_procedure_call) service involving a high-level API to communicate and extend Terraform's functionality.

To start a new plugin you only need these few lines of code in your [`main`](https://github.com/Burmuley/terraform-provider-sqlite/blob/v0.1.0/main.go) function:
```golang
func main() {
  plugin.Serve(&plugin.ServeOpts{
    ProviderFunc: sqlite.Provider,
  })
}
```

This will initiate an [`RPC`](https://en.wikipedia.org/wiki/Remote_procedure_call) service passing provider schema configuration function, which is explained in next chapters of the article.

## Terraform Schema vs HCL

Every HCL statement you write in `*.tf` files will eventually be parsed by Terraform Code and returned to the plugin (provider) as [`schema.ResourceData`](https://github.com/hashicorp/terraform-plugin-sdk/blob/main/helper/schema/resource_data.go#L22-L41) object validated against [`schema`](https://www.terraform.io/docs/extend/schemas/index.html) you define for each of the provider components. Then the plugin itself will run corresponding action functions (also defined in the schema) to achieve the desired state and communicate back with the results.

**Note:** If any errors occurs on any of functions, the plugin API assumes you collect all the information and return to the Terraform Core as slice of [`Diagnostic`](https://github.com/hashicorp/terraform-plugin-sdk/blob/main/diag/diagnostic.go#L44-L76) objects from [`diag`](https://github.com/hashicorp/terraform-plugin-sdk/tree/main/diag) package. No panics or `error` returns should be done within your provider's functions facing the Terraform Plugin API.

For example, for the following resource definition:
```hcl
resource "provider_resource" "important_resource" {
  name = "TREASURE"
}
```

Terraform will validate it against the following schema:
```golang
schema.Resource{
  Schema: map[string]*schema.Schema{
    "name": {
      Type:        schema.TypeString,
      Required:    true,
      ForceNew:    true,
    }
  },

  ...
}
```

As you can see, each field within the resource configuration is just a key of a map of type `map[string]*schema.Schema`. Details on what data types are supported you can find on [`attributes and types`](https://www.terraform.io/docs/extend/schemas/schema-types.html) page in the official docs.

Let's design our final HCL structures for configuring the provider itself and creating resources.

## Making up provider schema

For my case provider configuration is very simple and should only contain the path to the database file the provider is going to manage. This will be more than enough to initialize our SQLite database connection and create the database itself if it does not exists to the moment we run `terraform apply`.

```hcl
provider "sqlite" {
  path = "<path to database>"
}
```

How Terraform schema should look like for this provider?
Pretty simple (at least in the beginning):
```golang
func Provider() *schema.Provider {
  return &schema.Provider{
    Schema: map[string]*schema.Schema{
      "path": {
        Type:        schema.TypeString,
        Required:    true,
        DefaultFunc: schema.EnvDefaultFunc("SQLITE_DB_PATH", nil),
      },
    },
    ResourcesMap: map[string]*schema.Resource{
      "sqlite_table": resourceTable(),
      "sqlite_index": resourceIndex(),
    },
    ConfigureContextFunc: providerConfigure,
    DataSourcesMap:       map[string]*schema.Resource{},
    ProviderMetaSchema:   map[string]*schema.Schema{},
  }
}
```

Let's briefly walk through `schema.Provider` fields we use here:
* `Schema`: is a `map` containing configuration of the provider, where keys are the configuration fields names and values are [`schema.Schema`](https://github.com/hashicorp/terraform-plugin-sdk/blob/main/helper/schema/schema.go#L37-L245) structs defining corresponding [`internal type`](https://www.terraform.io/docs/extend/schemas/schema-types.html) of the field. It can be omitted if your provider does not have any configuration fields.

  Once the provider definition got loaded Terraform validates its configurations against this schema to check for consistency and defined [`behavioral attributes`](https://www.terraform.io/docs/extend/schemas/schema-behaviors.html).
* `ResourcesMap`: this is another `map` holding information about what resources this provider can manage. As always, keys are resources names (the ones you define in your HCL files right after keyword `resource`) and values are structs of type [`schema.Resource`](https://github.com/hashicorp/terraform-plugin-sdk/blob/main/helper/schema/resource.go#L44-L244).
  
  In this particular case they're defined as a function calls (`resourceTable()` and `resourceIndex()`) to simplify code and increase its readability. I will talk about these functions in the next chapter.
* `ConfigureContextFunc`: this is the most important setting for provider schema as it defines how all the initialization magic happens.

  Here you need to provide a function of the following interface: `func(ctx context.Context, d *schema.ResourceData) (interface{}, diag.Diagnostics)`.
  
  This function will be invoked by Terraform during provider initialization and pass parsed (and validated) provider configuration as `d` of type `*schema.ResourceData`.

  Inside the function you need to add a code to initialize an entity that will be used by all other functions to create/update/read/delete resources for the service your provider is aimed to work with. For REST API that will be a connection pool with a prepared client (configured endpoints, valid authentication etc). In case of SQLite this is gonna be just an instance of `sqLiteWrapper` object with opened database file.

  The function need to return such entity along with any `diag.Diagnostic` objects and its [`code`](https://github.com/Burmuley/terraform-provider-sqlite/blob/v0.1.0/sqlite/provider.go) look like this:
  ```golang
  func providerConfigure(ctx context.Context, d *schema.ResourceData) (interface{}, diag.Diagnostics) {
    var err error
    var diags diag.Diagnostics

    dbPath := d.Get("path").(string)
    if len(dbPath) < 1 {
      diags = append(diags, diag.Diagnostic{
        Severity: diag.Error,
        Summary:  "parameter 'path' can not be empty",
      })
      return nil, diags
    }

    sqlW := NewSqLiteWrapper()
    sqlW.Open(dbPath)

    if err != nil {
      diags = append(diags, diag.Diagnostic{
        Severity: diag.Error,
        Summary:  fmt.Sprintf("error opening the database '%s'", dbPath),
        Detail:   fmt.Sprint(err),
      })
      return nil, diags
    }

    return sqlW, diags
  }
  ```

* `DataSourcesMap` and `ProviderMetaSchema`: These fields are not used in this example, but they can not be `nil`, so I just provided empty maps of the corresponding type.

Details on all the configuration fields available in [`schema.Provider`](https://github.com/hashicorp/terraform-plugin-sdk/blob/main/helper/schema/provider.go#L49-L96) struct from [`helper/schema`](https://github.com/hashicorp/terraform-plugin-sdk/tree/main/helper/schema) package you can always find in the official [repository](https://github.com/hashicorp/terraform-plugin-sdk/tree/main/helper/schema).

Here we come to the next breath taking step - defining the provider resources and its functionality! ;)

## Adding resources

You remember in previous chapter I defined a `map` exposing a list of resources provided by the plugin:
```golang
ResourcesMap: map[string]*schema.Resource{
  "sqlite_table": resourceTable(),
  "sqlite_index": resourceIndex(),
}
```

As you might guessed it, functions defined as values in the `map` simply return [`*schema.Resource`](https://github.com/hashicorp/terraform-plugin-sdk/blob/main/helper/schema/resource.go#L44-L244) structure and it's just a "helper" to avoid a mess in the code.

For our resource `sqlite_table` the HCL resource representation will look like:
```hcl
resource "sqlite_table" "test_table" {
  name = "users"

  column {
    name = "id"
    type = "INTEGER"
    constraints {
      not_null = true
      primary_key = true
    }
  }

  column {
    name = "name"
    type = "TEXT"
    constraints {
      not_null = true
    }
  }
}
```

Basically it has only two properties `name` and `column`, where `column` is a nested block which could have multiple instances (i.e. columns). In provider schema it will be represented as a list of `map`s with corresponding value holding such properties as `type` and `constraints`.

The final schema for this resource looks like:
```golang
func resourceTable() *schema.Resource {
  return &schema.Resource{
    Schema: map[string]*schema.Schema{
      "name": {
        Type:        schema.TypeString,
        Required:    true,
        ForceNew:    true,
      },
      "created": {
        Type:        schema.TypeString,
        Computed:    true,
      },
      "column": {
        Type:     schema.TypeList,
        Required: true,
        ForceNew: true,
        Elem: &schema.Resource{
          Schema: map[string]*schema.Schema{
            "name": {
              Type:        schema.TypeString,
              Required:    true,
              ForceNew:    true,
            },
            "type": {
              Type:        schema.TypeString,
              Required:    true,
              ForceNew:    true,
            },
            "constraints": {
              Type:        schema.TypeList,
              MaxItems:    1,
              Optional:    true,
              ForceNew:    true,
              Elem: &schema.Resource{
                Schema: map[string]*schema.Schema{
                  "primary_key": {
                    Type:     schema.TypeBool,
                    Optional: true,
                    ForceNew: true,
                    Default:  false,
                  },
                  "not_null": {
                    Type:     schema.TypeBool,
                    Optional: true,
                    ForceNew: true,
                    Default:  false,
                  },
                  "default": {
                    Type:     schema.TypeString,
                    Optional: true,
                    ForceNew: true,
                    Default:  nil,
                  },
                },
              },
            },
          },
        },
      },
    },
    CreateContext: resourceTableCreate,
    ReadContext:   resourceTableRead,
    DeleteContext: resourceTableDelete,
  }
}
```

In this schema:
* `created`: is just a computed field stored in the state, not going to be used in this version of the provider
* `name`: the table name that will be put after `CREATE TABLE` words in the generated statement, it's a simple string, nothing special
* `column`: here we have some interesting setup. For the `Type` of `schema.TypeList` we should define the particular type of the list elements in the `Elem` property. And in this case that will be an underlying `map` with al the fields you saw in the `HCL` above.
  
  The funny thing is that if you'd define that list as the following:
  ```golang
  "column": {
        Type:     schema.TypeList,
        Elem: schema.TypeString
        }
  ```
  The the HCL statement for the `column` would be like:
  ```hcl
  resource "sqlite_table" "test_table" {
    ...
    column = ["one", "two", ...]
    ...
  }
  ```

  So, defining an [`Aggregate type`](https://www.terraform.io/docs/extend/schemas/schema-types.html#aggregate-types) in `Elem` makes your entity a nested block.

  The internal structure of `constraints` field has the same approach as the "main" schema for the resource.

Also want to point you to one important field `ForceNew` in almost all properties defined within the schema. This field sets behavior so if the field has changed since previous `terraform apply` run then the resource should be deleted and then created again with new configuration. Since this version of the provider is not intended to support any SQL schema updates the resource schema has `true` value for all of the user-facing properties.

There are some other interesting properties we need to pay attention to: `CreateContext`, `ReadContext` and `DeleteContext`. In these fields you can define the particular functions that will be invoked by Terraform on the corresponding stage like `create` or `delete`.

The function in the value for these fields should implement the following interface: `func(ctx context.Context, d *schema.ResourceData, m interface{}) diag.Diagnostics`.

Here parameters of the function are:
* `d *schema.ResourceData`: purpose of this parameter is vary on the stage and can be used to read or write values.
  
  * on `Create` stage this parameter holds parsed and validated against schema resource definition, so you can read values from it to make up a request to an external system (generate SQL statement in our case) and create it
  * on `Delete` stage the parameter holds the resource configuration stored in Terraform state, so you can read values required to perform deletion of the resource in the remote system
  * on `Read` stage (invoked when you run `terraform apply/plan` **and** module has non empty state) the parameter has empty structure and your task is to read the current resource state from the remote system and put it into the parameter so Terraform later could calculate the `diff` and decide what to do with the resource

* `m interface{}`: this is the pointer to an object returned by `ConfigureContextFunc` defined in the provider schema we defined earlier

To avoid inflating of the contents of this article here I'll only provide an example of short function that deletes tables (defined in `DeleteContext` property above):
```golang
func resourceTableDelete(ctx context.Context, d *schema.ResourceData, m interface{}) diag.Diagnostics {
  c := m.(*sqLiteWrapper)
  query := fmt.Sprintf("DROP TABLE %s;", d.Id())
  _, err := c.Exec(query)
  if err != nil {
    return diag.FromErr(err)
  }
  d.SetId("")
  return diag.Diagnostics{}
}
```

The rest of the code you can find in the [`sqlite/resource_table.go `](https://github.com/Burmuley/terraform-provider-sqlite/blob/v0.1.0/sqlite/resource_table.go).

The code for managing `sqlite_index` resources is placed in [`sqlite/resource_index.go `](https://github.com/Burmuley/terraform-provider-sqlite/blob/v0.1.0/sqlite/resource_index.go) and is very similar to the code for tables.

## Code structure

I held this part till the near end to summarize the "product" structure after the all boring stuff.

Eventually the root directory contains only `main.go` file with a few of contents aimed to initialize and start and RPC service, and the rest of the provider logic I placed into a nested package `sqlite`.


```shell
.
├── build.sh
├── example
│   ├── main.tf
│   └── provider.tf
│
├── sqlite
│    ├── provider.go
│    ├── resource_index.go
│    ├── resource_table.go
│    ├── sqlite.go
│    ├── templates.go
│    └── templates_helpers.go
│
├── main.go
└── version.txt
```

The file in `sqlite` directory:
* `provider.go`: holds provider schema and initialization function
* `resource_index.go` & `resource_table.go`: holds corresponding resources schema and `action` functions to provide
* `sqlite.go`: contains implementation of the `sqLiteWrapper` type used in the rest of code for SQLite resources management
* `templates.go` & `templates_helpers.go`: this is just code with templates for SQL statements generation

Feel free to inspect allt he sources of the first version [`here`](https://github.com/Burmuley/terraform-provider-sqlite/tree/v0.1.0).

## Building and local testing

For custom providers Terraform has some mechanisms to load plugins from local sources, you not necessary need to publish your binaries anywhere when you just need to play with something.

So, as I mentioned before, any Terraform plugin is a simple Go binary and it can be built with a regular command:
```shell
go build -o terraform-provider-sqlite
```

Then you just need to move this fresh binary to a special cache directory Terraform used to scan when discovering providers (code works for MacOS and Linux):
```shell
mkdir -p ~/.terraform.d/plugins/burmuley.com/edu/sqlite/0.1/darwin_amd64
mv terraform-provider-sqlite ~/.terraform.d/plugins/burmuley.com/edu/sqlite/0.1/darwin_amd64
```

Now you can simply define provider source in your local HCL code:
```hcl
terraform {
  required_providers {
    sqlite = {
      version = "0.1"
      source = "burmuley.com/edu/sqlite"
    }
  }
}
```

## What's next?

I'm not finished on developing this provider and in the next version I gonna implement some new features:
* SQL schema updates (that seems to me very complicated at least for now)
* SQLite resource import into Terraform state

Keep in touch ;)

