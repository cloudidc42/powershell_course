# Part 25: Full Stack App - สร้าง Blog API

> **ระดับ**: 🟠 Advanced | **เวลา**: ~6 ชั่วโมง

---

## Project Structure

```
blog-api/
├── server.ps1        # จุดเริ่มต้น
├── data/
│   └── blog.db          # SQLite database
├── lib/
│   ├── Database.ps1     # DB layer
│   ├── Auth.ps1         # JWT auth
│   ├── Router.ps1       # HTTP router
│   └── Models.ps1       # Data models
├── routes/
│   ├── auth.ps1
│   ├── posts.ps1
│   └── comments.ps1
└── public/
    ├── index.html
    ├── style.css
    └── app.js
```

---

## 1. Database Layer

```powershell
# lib/Database.ps1
class BlogDatabase {
    hidden [hashtable]$_store  # In-memory for demo
    hidden [int]$_postId = 0
    hidden [int]$_userId = 0
    hidden [int]$_commentId = 0
    
    BlogDatabase() {
        $this._store = @{
            users    = @{}
            posts    = @{}
            comments = @{}
        }
        $this._Seed()
    }
    
    hidden [void] _Seed() {
        $adminId = $this.CreateUser(@{
            username = 'admin'
            password = 'hashed_admin_pass'
            email    = 'admin@example.com'
            role     = 'admin'
        })
        
        $post = $this.CreatePost(@{
            title   = 'Welcome to the Blog'
            content = 'This is the first post powered by PowerShell!'
            userId  = $adminId
            tags    = @('powershell','blog','intro')
        })
    }
    
    # Users
    [string] CreateUser([hashtable]$data) {
        $id = [guid]::NewGuid().ToString('N').Substring(0,8)
        $this._store.users[$id] = @{
            id       = $id
            username = $data.username
            password = $data.password  # ควร hash
            email    = $data.email
            role     = $data.role ?? 'user'
            created  = Get-Date -Format 'o'
        }
        return $id
    }
    
    [hashtable] GetUserByUsername([string]$username) {
        return $this._store.users.Values | Where-Object { $_.username -eq $username } | Select-Object -First 1
    }
    
    # Posts
    [string] CreatePost([hashtable]$data) {
        $id = [guid]::NewGuid().ToString('N').Substring(0,8)
        $this._store.posts[$id] = @{
            id      = $id
            title   = $data.title
            content = $data.content
            userId  = $data.userId
            tags    = $data.tags ?? @()
            created = Get-Date -Format 'o'
            updated = $null
            views   = 0
        }
        return $id
    }
    
    [hashtable[]] GetPosts([int]$page = 1, [int]$perPage = 10, [string]$tag = '') {
        $posts = $this._store.posts.Values | Sort-Object created -Descending
        if ($tag) { $posts = $posts | Where-Object { $_.tags -contains $tag } }
        return @($posts | Select-Object -Skip (($page-1)*$perPage) -First $perPage)
    }
    
    [hashtable] GetPost([string]$id) {
        $post = $this._store.posts[$id]
        if ($post) { $post.views++ }
        return $post
    }
    
    [hashtable] UpdatePost([string]$id, [hashtable]$data) {
        if (!$this._store.posts[$id]) { return $null }
        $post = $this._store.posts[$id]
        if ($data.title)   { $post.title = $data.title }
        if ($data.content) { $post.content = $data.content }
        if ($data.tags)    { $post.tags = $data.tags }
        $post.updated = Get-Date -Format 'o'
        return $post
    }
    
    [bool] DeletePost([string]$id) {
        if (!$this._store.posts[$id]) { return $false }
        $this._store.posts.Remove($id)
        # ลบ comments ที่เกี่ยวข้อง
        $toRemove = $this._store.comments.Values | Where-Object { $_.postId -eq $id }
        $toRemove | ForEach-Object { $this._store.comments.Remove($_.id) }
        return $true
    }
}

$db = [BlogDatabase]::new()
```

---

## 2. Route Handlers

```powershell
# routes/posts.ps1
function Register-PostRoutes {
    param([WebApp]$App, [BlogDatabase]$Db, [string]$JwtSecret)
    
    # GET /api/posts?page=1&tag=powershell
    $App.Get('/api/posts', {
        param($ctx)
        $page    = [int]($ctx.QueryParams['page'] ?? 1)
        $perPage = [int]($ctx.QueryParams['per_page'] ?? 10)
        $tag     = $ctx.QueryParams['tag'] ?? ''
        
        $posts = $using:Db.GetPosts($page, $perPage, $tag)
        $total = $using:Db._store.posts.Count
        
        $ctx.Json(@{
            data  = $posts
            meta  = @{ page=$page; perPage=$perPage; total=$total }
        })
    })
    
    # GET /api/posts/:id
    $App.Get('/api/posts/:id', {
        param($ctx)
        $id   = $ctx.RouteParams['id']
        $post = $using:Db.GetPost($id)
        if (!$post) { $ctx.NotFound("Post '$id' not found"); return }
        $ctx.Json($post)
    })
    
    # POST /api/posts (protected)
    $App.Post('/api/posts', {
        param($ctx)
        
        # Auth check
        $claims = Invoke-JwtMiddleware $ctx $using:JwtSecret
        if (!$claims) { return }
        
        $data = $ctx.ReadJson()
        if (!$data.title -or !$data.content) {
            $ctx.BadRequest('title and content are required')
            return
        }
        
        $id = $using:Db.CreatePost(@{
            title   = $data.title
            content = $data.content
            userId  = $claims.sub
            tags    = $data.tags ?? @()
        })
        
        $post = $using:Db.GetPost($id)
        $ctx.Json($post, 201)
    })
    
    # PUT /api/posts/:id (protected)
    $App.Put('/api/posts/:id', {
        param($ctx)
        $claims = Invoke-JwtMiddleware $ctx $using:JwtSecret
        if (!$claims) { return }
        
        $id   = $ctx.RouteParams['id']
        $post = $using:Db.GetPost($id)
        if (!$post) { $ctx.NotFound("Post '$id' not found"); return }
        
        # Only author or admin can edit
        if ($post.userId -ne $claims.sub -and $claims.role -ne 'admin') {
            $ctx.Json(@{error='Forbidden'}, 403)
            return
        }
        
        $data    = $ctx.ReadJson()
        $updated = $using:Db.UpdatePost($id, @{ title=$data.title; content=$data.content; tags=$data.tags })
        $ctx.Json($updated)
    })
    
    # DELETE /api/posts/:id (protected)
    $App.Delete('/api/posts/:id', {
        param($ctx)
        $claims = Invoke-JwtMiddleware $ctx $using:JwtSecret
        if (!$claims) { return }
        
        $id   = $ctx.RouteParams['id']
        $post = $using:Db._store.posts[$id]
        if (!$post) { $ctx.NotFound("Post '$id' not found"); return }
        
        if ($post.userId -ne $claims.sub -and $claims.role -ne 'admin') {
            $ctx.Json(@{error='Forbidden'}, 403); return
        }
        
        $using:Db.DeletePost($id)
        $ctx.Json(@{message='Post deleted'})
    })
}

# JWT auth helper
function Invoke-JwtMiddleware {
    param($ctx, $secret)
    $auth = $ctx.Request.Headers['Authorization']
    if (!$auth -or !$auth.StartsWith('Bearer ')) {
        $ctx.Json(@{error='Unauthorized'}, 401)
        return $null
    }
    $token = $auth.Substring(7)
    $claims = Test-JwtToken $token $secret
    if (!$claims) {
        $ctx.Json(@{error='Invalid or expired token'}, 401)
        return $null
    }
    return $claims
}
```

---

## 3. server.ps1

```powershell
# server.ps1 - entry point
#Requires -Version 7.0

. '.\lib\Router.ps1'
. '.\lib\Auth.ps1'
. '.\lib\Models.ps1'
. '.\routes\posts.ps1'
. '.\routes\auth.ps1'

$JWT_SECRET = $env:JWT_SECRET ?? 'dev-secret-change-in-production'
$PORT       = [int]($env:PORT ?? 8080)
$db         = [BlogDatabase]::new()
$app        = [WebApp]::new()

# Middleware
$app.Use({param($ctx)  # Logging
    $m = $ctx.Request.HttpMethod
    $p = $ctx.Request.Url.AbsolutePath
    Write-Host "$(Get-Date -Format 'HH:mm:ss') $m $p" -ForegroundColor DarkGray
})

$app.Use({param($ctx)  # CORS
    $ctx.Response.AddHeader('Access-Control-Allow-Origin', '*')
    $ctx.Response.AddHeader('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE,OPTIONS,PATCH')
    $ctx.Response.AddHeader('Access-Control-Allow-Headers', 'Content-Type,Authorization')
    if ($ctx.Request.HttpMethod -eq 'OPTIONS') {
        $ctx.Response.StatusCode = 204; $ctx.Response.OutputStream.Close()
    }
})

# Register routes
Register-AuthRoutes  $app $db $JWT_SECRET
Register-PostRoutes  $app $db $JWT_SECRET

# ส่ง static files
$app.Get('/', { param($ctx)
    $html = Get-Content 'public/index.html' -Raw
    $ctx.Html($html) 
})
$app.Get('/static/:file', { param($ctx)
    Invoke-StaticFile $ctx 'public'
})

Write-Host "» Blog API v1.0" -ForegroundColor Green
Write-Host "  http://localhost:$PORT/api/posts" -ForegroundColor Cyan
$app.Listen($PORT)
```

---

**ก่อนหน้า ← [Part 24](Part-24.md) | ต่อไป → [Part 26: Pester Testing](Part-26.md)**
