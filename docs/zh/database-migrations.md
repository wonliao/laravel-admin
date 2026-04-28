# 数据库迁移

本文说明 `laravel-admin` 项目中的数据库迁移策略，包括业务表 migration、内置 `admin_*` 表 migration、发布与执行流程、回滚、修改表结构、索引、软删除、关联表与环境注意事项。

`laravel-admin` 使用 Laravel migration，不提供独立的数据库迁移系统。所有迁移仍由宿主 Laravel 应用执行：

```bash
php artisan migrate
```

## migration 的职责

Migration 用来描述数据库结构的版本化变更，应该负责：

- 创建表
- 修改表
- 删除表
- 增加/修改/删除字段
- 增加/删除索引
- 创建 pivot table
- 增加软删除字段
- 记录可回滚的结构变化

不建议在 migration 中写大量业务数据。初始化后台用户、角色、权限、菜单这类数据，应放到 seeder 或专门的初始化命令。

## laravel-admin 内置迁移

`laravel-admin` 内置迁移文件位于：

```text
database/migrations/2016_01_04_173148_create_admin_tables.php
```

发布到宿主 Laravel 应用后，会创建默认后台表：

- `admin_users`
- `admin_roles`
- `admin_permissions`
- `admin_menu`
- `admin_operation_log`
- `admin_role_users`
- `admin_role_permissions`
- `admin_user_permissions`
- `admin_role_menu`

这些表名来自 `config/admin.php`：

```php
'database' => [
    'connection' => '',
    'users_table' => 'admin_users',
    'roles_table' => 'admin_roles',
    'permissions_table' => 'admin_permissions',
    'menu_table' => 'admin_menu',
    'operation_log_table' => 'admin_operation_log',
    'user_permissions_table' => 'admin_user_permissions',
    'role_users_table' => 'admin_role_users',
    'role_permissions_table' => 'admin_role_permissions',
    'role_menu_table' => 'admin_role_menu',
],
```

内置 migration 的 connection 由以下逻辑决定：

```php
public function getConnection()
{
    return config('admin.database.connection') ?: config('database.default');
}
```

如果 `admin.database.connection` 为空，就使用 Laravel 默认数据库连接。

## 发布 migration

安装时通常执行：

```bash
php artisan vendor:publish --provider="Encore\Admin\AdminServiceProvider"
```

或使用：

```bash
php artisan admin:publish
```

发布后会把 package 中的 migration 复制到宿主应用：

```text
database/migrations/
└── 2016_01_04_173148_create_admin_tables.php
```

如果需要覆盖已经发布过的文件：

```bash
php artisan admin:publish --force
```

使用 `--force` 前要先确认本地 migration 没有项目自定义修改，否则会被覆盖。

## 执行 migration

执行所有未执行过的 migration：

```bash
php artisan migrate
```

查看 migration 状态：

```bash
php artisan migrate:status
```

重新创建所有表并重新执行 migration：

```bash
php artisan migrate:fresh
```

开发环境中若要同时执行 seed：

```bash
php artisan migrate:fresh --seed
```

`admin:install` 内部会先执行：

```bash
php artisan migrate
```

然后在后台用户表为空时执行 `AdminTablesSeeder`，创建默认 `admin/admin` 与基础权限菜单。

## 创建业务表 migration

使用 Laravel 命令创建 model 与 migration：

```bash
php artisan make:model Post -m
```

示例：

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('body')->nullable();
            $table->boolean('published')->default(false);
            $table->timestamp('published_at')->nullable();
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('posts');
    }
};
```

运行：

```bash
php artisan migrate
```

然后建立后台 CRUD：

```bash
php artisan admin:make PostController --model=App\\Models\\Post
```

## 修改既有表结构

不要直接修改已经在其他环境执行过的 migration。正确做法是新增一个 migration。

示例：给 `posts` 表新增 `summary` 字段。

```bash
php artisan make:migration add_summary_to_posts_table --table=posts
```

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::table('posts', function (Blueprint $table) {
            $table->string('summary')->nullable()->after('title');
        });
    }

    public function down()
    {
        Schema::table('posts', function (Blueprint $table) {
            $table->dropColumn('summary');
        });
    }
};
```

执行：

```bash
php artisan migrate
```

如果后台 Form 需要编辑这个字段，同步更新 controller：

```php
$form->text('summary', '摘要');
```

如果 Grid 需要展示：

```php
$grid->column('summary', '摘要')->limit(40);
```

## 字段类型建议

常见字段与 Form/Grid 对应：

| 数据库字段 | Laravel migration | Form 字段 | Grid 显示 |
| --- | --- | --- | --- |
| 短文本 | `$table->string('title')` | `$form->text('title')` | `$grid->column('title')` |
| 长文本 | `$table->text('body')` | `$form->textarea('body')` | `$grid->column('body')->limit(50)` |
| 数字 | `$table->integer('sort')` | `$form->number('sort')` | `$grid->column('sort')->sortable()` |
| 布尔 | `$table->boolean('published')` | `$form->switch('published')` | `$grid->column('published')->bool()` |
| 日期时间 | `$table->timestamp('published_at')` | `$form->datetime('published_at')` | `$grid->column('published_at')` |
| JSON | `$table->json('meta')` | `$form->keyValue('meta')` 或自定义字段 | `$grid->column('meta')` |
| 图片路径 | `$table->string('cover')` | `$form->image('cover')` | `$grid->column('cover')->image()` |
| 文件路径 | `$table->string('file')` | `$form->file('file')` | `$grid->column('file')` |

表单字段并不自动创建数据库列。必须先在 migration 中定义字段，再在 Form/Grid/Show 中使用。

## 索引与查询性能

后台列表经常会按筛选器查询。凡是经常用于过滤、排序、关联的字段，都应考虑索引。

示例：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->index('published');
    $table->index('published_at');
    $table->index(['user_id', 'created_at']);
});
```

对应 Grid：

```php
$grid->model()->orderBy('created_at', 'desc');

$grid->filter(function (Grid\Filter $filter) {
    $filter->equal('published', '发布状态');
    $filter->between('created_at', '创建时间')->datetime();
});
```

如果 Grid 使用关联字段，应确保外键字段有索引：

```php
$table->foreignId('user_id')->index();
```

## 软删除

如果希望后台删除记录后可以恢复，使用 Laravel soft deletes。

Migration：

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->softDeletes();
    $table->timestamps();
});
```

Model：

```php
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;
}
```

如果是给既有表新增软删除：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->softDeletes();
});
```

回滚：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->dropSoftDeletes();
});
```

## 关联表与 pivot table

### belongsTo

文章属于一个用户：

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->index();
    $table->string('title');
    $table->timestamps();
});
```

Form：

```php
$form->select('user_id', '作者')
    ->options(User::all()->pluck('name', 'id'));
```

### belongsToMany

文章与标签多对多：

```php
Schema::create('post_tag', function (Blueprint $table) {
    $table->unsignedBigInteger('post_id');
    $table->unsignedBigInteger('tag_id');
    $table->primary(['post_id', 'tag_id']);
});
```

Model：

```php
class Post extends Model
{
    public function tags()
    {
        return $this->belongsToMany(Tag::class, 'post_tag');
    }
}
```

Form：

```php
$form->multipleSelect('tags', '标签')
    ->options(Tag::all()->pluck('name', 'id'));
```

字段名 `tags` 必须对应 Eloquent relation 方法名。

## 外键约束策略

`laravel-admin` 内置 `admin_*` migration 没有创建数据库 foreign key constraint，只创建索引。这种设计让安装、回滚、跨数据库兼容更简单，但引用完整性主要依赖应用逻辑。

业务表可按项目要求选择：

### 使用数据库外键

```php
$table->foreignId('user_id')
    ->constrained()
    ->cascadeOnDelete();
```

优点是数据库层能保证一致性。风险是删除父表记录会影响子表，回滚和测试数据清理也更严格。

### 只建索引，不建外键

```php
$table->unsignedBigInteger('user_id')->index();
```

优点是灵活、容易导入和清理数据。风险是可能产生孤儿数据，需要应用层处理。

选择哪种策略应与项目整体数据库规范一致。

## 多数据库连接

如果后台内置表要放到独立连接，可以配置：

```php
'database' => [
    'connection' => 'admin',
],
```

同时在 Laravel `config/database.php` 中定义 `admin` connection。

业务 model 也可以指定 connection：

```php
class Post extends Model
{
    protected $connection = 'mysql';
}
```

注意：

- `config('admin.database.connection')` 只影响内置 admin 表。
- 业务表使用哪个连接由业务 model 或 Laravel 默认连接决定。
- 跨连接关联、事务与 join 需要谨慎处理。

## 回滚与重建

回滚最近一批 migration：

```bash
php artisan migrate:rollback
```

回滚多步：

```bash
php artisan migrate:rollback --step=3
```

重置所有 migration 后再执行：

```bash
php artisan migrate:refresh
```

删除所有表并重新执行：

```bash
php artisan migrate:fresh
```

开发环境可以使用 `migrate:fresh`。生产环境不要随意执行，因为它会删除所有表。

## 与 admin:install 的关系

`admin:install` 会调用：

```php
$this->call('migrate');
```

然后检查后台用户表：

```php
$userModel = config('admin.database.users_model');

if ($userModel::count() == 0) {
    $this->call('db:seed', [
        '--class' => Encore\Admin\Auth\Database\AdminTablesSeeder::class,
    ]);
}
```

因此：

- 首次安装时会创建 `admin_*` 表并 seed 默认账号。
- 如果后台用户表已有数据，不会重复 seed 默认账号。
- 如果 `app/Admin` 已存在，不会覆盖应用侧后台目录。

## migration 与 seed 的分工

建议分工：

| 类型 | 放在 migration | 放在 seeder |
| --- | --- | --- |
| 建表 | 是 | 否 |
| 加字段 | 是 | 否 |
| 加索引 | 是 | 否 |
| 初始化后台账号 | 否 | 是 |
| 初始化角色权限 | 否 | 是 |
| 初始化菜单 | 否 | 是 |
| 大量业务示例数据 | 否 | 是 |

原因：

- migration 应该描述结构。
- seeder 应该描述数据。
- 结构变更更需要可回滚。
- 数据初始化经常会按环境不同而变化。

## 开发、测试、生产环境建议

### 开发环境

可使用：

```bash
php artisan migrate:fresh --seed
```

适合快速重建数据库，但会删除所有表。

### 测试环境

推荐使用独立测试数据库，并在测试前重建：

```bash
php artisan migrate:fresh --env=testing
```

`laravel-admin` 自身测试默认需要可连接的数据库；如果使用 MySQL，要确认测试数据库与连接参数存在。

### 生产环境

生产环境建议：

1. 先备份数据库。
2. 先在 staging 环境执行 migration。
3. 检查 migration 是否可回滚。
4. 避免在高峰期执行大表结构变更。
5. 避免使用 `migrate:fresh`、`migrate:refresh`。
6. 对大表新增索引或修改字段时，评估锁表风险。

## 常见变更范例

### 新增 nullable 字段

```php
Schema::table('posts', function (Blueprint $table) {
    $table->string('subtitle')->nullable()->after('title');
});
```

### 新增唯一字段

```php
Schema::table('posts', function (Blueprint $table) {
    $table->string('slug')->unique()->after('title');
});
```

如果既有表已有数据，先确认每行都有唯一 slug，否则 migration 会失败。

### 修改字段长度

需要 `doctrine/dbal` 支援。`laravel-admin` 的 composer 依赖已经包含 `doctrine/dbal`。

```php
Schema::table('posts', function (Blueprint $table) {
    $table->string('title', 500)->change();
});
```

### 删除字段

```php
Schema::table('posts', function (Blueprint $table) {
    $table->dropColumn('subtitle');
});
```

删除字段会丢数据。生产环境执行前应确认备份与回滚策略。

### 新增 JSON 字段

```php
Schema::table('posts', function (Blueprint $table) {
    $table->json('meta')->nullable();
});
```

Form 可使用：

```php
$form->keyValue('meta', '扩展信息');
```

## 排查入口

### 执行 admin:install 后没有 admin 表

检查：

- 是否发布了 migration。
- `php artisan migrate:status` 是否显示 migration 已执行。
- `config('admin.database.connection')` 是否指向了不同数据库。
- 当前 `.env` 的 `DB_*` 是否正确。

### 修改 migration 后没有生效

已经执行过的 migration 不会重复执行。应新增 migration，或在开发环境执行：

```bash
php artisan migrate:fresh
```

生产环境不要用 `migrate:fresh`。

### 后台 Form 字段保存时报 SQL unknown column

说明 Form 字段对应的数据库列不存在。检查：

- migration 是否定义字段。
- migration 是否已执行。
- model 是否使用了正确 table/connection。
- 字段名是否拼写一致。

### 发布 migration 后表名不是预期

内置 migration 使用 `config('admin.database.*_table')`。如果要改内置表名，应先改 `config/admin.php`，再执行 migration。已经执行后再改表名，需要额外写 rename migration 或重建数据库。

### 回滚失败

检查 `down()` 是否完整反向操作：

- `up()` 创建表，`down()` 应 drop table。
- `up()` 新增字段，`down()` 应 drop column。
- `up()` 新增索引，`down()` 应 drop index。
- `up()` 新增外键，`down()` 应先 drop foreign key。

## 检查清单

新增或修改数据库结构前，建议确认：

- migration 文件名与目标表名清楚。
- `up()` 与 `down()` 都实现。
- 字段类型与 Form/Grid 用法匹配。
- 经常筛选、排序、关联的字段有索引。
- 外键策略符合项目规范。
- 生产环境大表变更已评估锁表风险。
- seed 数据不写在 migration 中。
- `config/admin.php` 中 admin 内置表名与 connection 符合预期。
