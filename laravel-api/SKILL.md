---
name: laravel-api
description: Conventions for building REST APIs with Laravel. Apply this skill when creating or modifying API resources (controllers, form requests, resources, policies, migrations, factories, seeders, tests) in a Laravel project. Based on the book "Laravel REST APIs: A Practical Guide".
---

# Laravel REST API Conventions

This skill describes the conventions used in the book *Laravel REST APIs: A Practical Guide* and the companion project [BookShelf](https://github.com/gitantonio/bookshelf).

Apply these conventions when the user asks you to create or modify API resources, controllers, validation, authorization, or tests in a Laravel project. When conventions already present in the user's codebase differ from the ones listed here, prefer the user's conventions.

---

## Checklist for a new API resource

When adding a new resource (e.g. `Publisher`, `Tag`), you generally need:

1. **Model** with explicit `$fillable` and relationships.
2. **Migration** with correct column types, nullability and foreign keys via `constrained()`.
3. **Factory** with realistic data via Faker.
4. **Seeder** registered in `DatabaseSeeder`.
5. **API Resource** with explicit fields (never `parent::toArray()` nor returning the model directly).
6. **Controller** with authorization and validation.
7. **Form Request** when validation has several rules or is reused; inline `$request->validate([...])` for simple cases.
8. **Routes**: public reads outside `auth:sanctum`, writes inside it.
9. **Policy** when the resource has an owner (e.g. a `user_id` column) or any non-trivial authorization.
10. **Pest tests** covering CRUD, validation, authentication and authorization.

---

## Model

```php
class Book extends Model
{
    use HasFactory;

    // $fillable contains ONLY fields the client can set.
    // Never include user_id or computed fields.
    protected $fillable = [
        'title',
        'isbn',
        'description',
        'publication_year',
        'language',
        'pages',
        'author_id',
        'publisher_id',
    ];

    // Singular names for belongsTo, plural for hasMany/belongsToMany.
    public function author()
    {
        return $this->belongsTo(Author::class);
    }

    public function genres()
    {
        return $this->belongsToMany(Genre::class);
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

Rules:

- `$fillable` NEVER contains `user_id`, computed fields or server-controlled fields.
- When a resource has an owner, set the `user_id` server-side in the controller.

---

## Migration

```php
public function up(): void
{
    Schema::create('books', function (Blueprint $table) {
        $table->id();

        // Required fields: no nullable()
        $table->string('title');
        $table->string('isbn')->unique();

        // Optional fields: nullable()
        $table->text('description')->nullable();

        $table->unsignedSmallInteger('publication_year');
        $table->char('language', 2)->default('en');
        $table->unsignedSmallInteger('pages')->nullable();

        // Foreign keys: constrained() + explicit delete behavior
        $table->foreignId('author_id')
              ->nullable()
              ->constrained()
              ->nullOnDelete();

        $table->foreignId('user_id')
              ->constrained()
              ->cascadeOnDelete();

        $table->timestamps();

        // Indexes on columns used in filters
        $table->index('publication_year');
        $table->index('language');
    });
}
```

Rules:

- `nullOnDelete()` for optional relations (child survives if parent is deleted).
- `cascadeOnDelete()` for strong ownership (child must die with parent).
- Add indexes on columns used in filters, sorting, or frequent `where` clauses.
- Use precise types: `unsignedSmallInteger` for years, `char(2)` for ISO codes, etc.

---

## Factory

```php
class BookFactory extends Factory
{
    public function definition(): array
    {
        return [
            'title' => fake()->sentence(3),
            'isbn' => fake()->unique()->isbn13(),
            'description' => fake()->paragraph(),
            'publication_year' => fake()->numberBetween(1960, 2026),
            'language' => fake()->randomElement(['en', 'it', 'fr', 'es']),
            'pages' => fake()->numberBetween(80, 900),
            'author_id' => Author::factory(),
            'user_id' => User::factory(),
        ];
    }
}
```

Factories set fields directly on the model and bypass mass assignment, so they can legitimately set `user_id` and other server-controlled fields.

---

## API Resource

```php
class BookResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'isbn' => $this->isbn,

            // Heavy/long fields only in detail view
            'description' => $this->when(
                $request->routeIs('books.show'),
                $this->description
            ),

            'publication_year' => $this->publication_year,
            'language' => $this->language,
            'pages' => $this->pages,

            // Computed fields
            'is_recent' => $this->publication_year >= now()->year - 2,

            // Relations only appear when eager loaded
            'author' => new AuthorResource(
                $this->whenLoaded('author')
            ),
            'genres' => GenreResource::collection(
                $this->whenLoaded('genres')
            ),

            // Dates always ISO 8601
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
        ];
    }
}
```

Rules:

- NEVER use `parent::toArray()`. The resource is the API contract: every field must be explicit.
- NEVER return the model directly from a controller.
- Use `whenLoaded()` for relations so they appear only when eager-loaded.
- Use `when($request->routeIs('resource.show'), ...)` for fields that should appear only in the detail view (description, long text, heavy aggregates).
- Dates always in ISO 8601 (`toISOString()`).

---

## Controller

```php
/**
 * @group Books
 */
class BookController extends Controller
{
    /**
     * @unauthenticated
     *
     * @queryParam page integer Page number. Example: 1
     * @queryParam per_page integer Results per page (max 100). Example: 15
     */
    public function index(Request $request)
    {
        $this->authorize('viewAny', Book::class);

        $books = Book::query()
            ->with(['author', 'genres'])
            ->paginate(15);

        return BookResource::collection($books);
    }

    /**
     * @authenticated
     */
    public function store(StoreBookRequest $request)
    {
        $this->authorize('create', Book::class);

        $book = $request->user()->books()->create(
            $request->validated()
        );

        $book->load(['author', 'genres']);

        return (new BookResource($book))
            ->response()
            ->setStatusCode(201);
    }

    /**
     * @unauthenticated
     */
    public function show(Book $book)
    {
        $this->authorize('view', $book);

        $book->load(['author', 'genres']);

        return new BookResource($book);
    }

    /**
     * @authenticated
     */
    public function update(UpdateBookRequest $request, Book $book)
    {
        $this->authorize('update', $book);

        $book->update($request->validated());
        $book->load(['author', 'genres']);

        return new BookResource($book);
    }

    /**
     * @authenticated
     */
    public function destroy(Book $book)
    {
        $this->authorize('delete', $book);

        $book->delete();

        return response()->json(null, 204);
    }
}
```

Rules:

- `$request->validated()` only. NEVER `$request->all()`.
- When the resource has an owner, create it via the user's relationship (`$request->user()->books()->create(...)`), not by assigning `user_id` manually.
- `$this->authorize()` before every operation, even reads (pairs with a Policy).
- Eager load relations before returning (avoids N+1 in resources that use `whenLoaded`).
- Status codes: 200 (index/show/update), 201 (store), 204 (destroy).
- Start query builders with `Model::query()` when chaining multiple methods, for clarity.

---

## Validation

Form Request for non-trivial validation (several rules, custom rules, or rules that need to be shared between store and update):

```php
class StoreBookRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'isbn' => [
                'required', 'string',
                new ValidIsbn13(),
                'unique:books',
            ],
            'description' => ['nullable', 'string', 'max:5000'],
            'publication_year' => [
                'required', 'integer',
                'min:1450',
                'max:' . (date('Y') + 1),
            ],
            'language' => ['sometimes', 'string', 'size:2'],
            'pages' => ['nullable', 'integer', 'min:1', 'max:10000'],
            'author_id' => [
                'nullable', 'integer',
                'exists:authors,id',
            ],
            'genre_ids' => ['sometimes', 'array'],
            'genre_ids.*' => ['integer', 'exists:genres,id'],
        ];
    }

    // Body parameters for Scribe documentation.
    // Put descriptions and examples here instead of
    // cluttering the controller PHPDoc with @bodyParam tags.
    public function bodyParameters(): array
    {
        return [
            'title' => [
                'description' => "The book's title.",
                'example' => 'The Name of the Rose',
            ],
            // ...
        ];
    }
}
```

For updates, switch `required` to `sometimes` and ignore the current record in `unique`:

```php
'isbn' => [
    'sometimes', 'string',
    new ValidIsbn13(),
    Rule::unique('books')->ignore($this->route('book')),
],
```

Inline validation for simple resources (few fields, no custom rules, used once):

```php
$validated = $request->validate([
    'first_name' => ['required', 'string', 'max:255'],
    'last_name' => ['required', 'string', 'max:255'],
    'bio' => ['nullable', 'string', 'max:5000'],
]);
```

Rules:

- Always **array syntax**, never pipe syntax.
- `exists:table,column` for every foreign key.
- `sometimes` when a field may be absent in updates but must be valid if present.
- `nullable` when a field can explicitly be set to `null` (e.g. clearing an optional column).
- Combine both (`['sometimes', 'nullable', ...]`) when the field is optional and can also be cleared.

---

## Routes

Routes live in `routes/api.php`. Split reads from writes explicitly instead of relying on `apiResource` when you need granular control.

```php
use App\Http\Controllers\BookController;

// Public reads
Route::get('books', [BookController::class, 'index']);
Route::get('books/{book}', [BookController::class, 'show']);

// Protected writes
Route::middleware('auth:sanctum')->group(function () {
    Route::post('books', [BookController::class, 'store']);
    Route::put('books/{book}', [BookController::class, 'update']);
    Route::patch('books/{book}', [BookController::class, 'update']);
    Route::delete('books/{book}', [BookController::class, 'destroy']);
});
```

Use `apiResource` only when ALL routes share the same auth configuration.

---

## Policy

```php
class BookPolicy
{
    // ?User (nullable) for public actions (guests allowed)
    public function viewAny(?User $user): bool
    {
        return true;
    }

    public function view(?User $user, Book $book): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return true;
    }

    public function update(User $user, Book $book): bool
    {
        return $user->id === $book->user_id;
    }

    public function delete(User $user, Book $book): bool
    {
        return $user->id === $book->user_id;
    }
}
```

Rules:

- `?User` (nullable) for actions where guests are allowed.
- `$user->id === $model->user_id` for ownership checks.
- Keep policies focused on ownership/permission. Business rules that depend on state (edit time windows, field-level limits) belong in the controller or in domain services, not in the policy.
- Create the policy even if all methods return `true`: it makes authorization explicit and easy to tighten later.

---

## Error handling

All API errors follow the same envelope:

```json
{
    "error": {
        "status": 422,
        "message": "The title field is required.",
        "details": {
            "title": ["The title field is required."]
        }
    }
}
```

For business-rule violations, throw a `BusinessException`:

```php
namespace App\Exceptions;

use Exception;

class BusinessException extends Exception
{
    public function __construct(string $message, int $code = 409)
    {
        parent::__construct($message, $code);
    }
}
```

Use it like this:

```php
if ($author->books()->exists()) {
    throw new BusinessException(
        'Cannot delete an author that has books.'
    );
}
```

Rules:

- Do NOT add a `statusCode` property to `BusinessException`: use the standard `$code` parameter, which the global exception handler reads.
- Do NOT catch `BusinessException` in the controller: let it bubble up to the exception handler that converts it to the error envelope.
- Default status code is `409 Conflict`; override with the second constructor argument when a different 4xx makes more sense.

---

## Pest tests

```php
use App\Models\Book;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('returns a paginated list of books', function () {
    Book::factory()->count(20)->create();

    $this->getJson('/api/books')
        ->assertOk()
        ->assertJsonCount(15, 'data')
        ->assertJsonPath('meta.total', 20);
});

it('creates a book', function () {
    $user = User::factory()->create();
    $author = Author::factory()->create();

    $this->actingAs($user)
        ->postJson('/api/books', [
            'title' => 'The Name of the Rose',
            'isbn' => '9780156001311',
            'publication_year' => 1980,
            'author_id' => $author->id,
        ])
        ->assertCreated()
        ->assertJsonPath('data.title', 'The Name of the Rose');
});

it('prevents unauthenticated users from creating books',
    function () {
    $this->postJson('/api/books', [])
        ->assertUnauthorized();
});

it('prevents updating another user\'s book', function () {
    $owner = User::factory()->create();
    $other = User::factory()->create();
    $book = Book::factory()
        ->create(['user_id' => $owner->id]);

    $this->actingAs($other)
        ->putJson("/api/books/{$book->id}", [
            'title' => 'Hacked',
        ])
        ->assertForbidden();
});
```

Rules:

- `actingAs($user)` to authenticate. Never log in via the API inside tests.
- Factories to create data. Never build arrays manually when a factory exists.
- One test, one behavior. If a test name contains "and", it probably should be two tests.
- Readable `it(...)` names that describe behavior, not methods (`it creates a book`, not `testStoreMethod`).
- Use `toEqual()` for numeric comparisons in `expect(...)` to avoid float/int/string type mismatches coming from the database.
- For validation errors (custom envelope), assert structure with `assertJsonStructure(['error' => ['details' => ['field']]])`, not `assertJsonValidationErrors()` which expects the Laravel default format.

---

## Scribe annotations

The API is documented with [Scribe](https://scribe.knuckles.wtf/laravel). Every controller and every method must carry explicit annotations. Do NOT rely on config defaults for authentication: mark every endpoint.

**Class level**:

```php
/**
 * @group Books
 */
class BookController extends Controller
```

**Method level** — a minimal template for a read endpoint:

```php
/**
 * List books
 *
 * @unauthenticated
 *
 * @queryParam page integer Page number. Example: 1
 * @queryParam per_page integer Results per page (max 100). Example: 15
 * @queryParam sort string Sort field (prefix with `-` for desc). Example: -publication_year
 * @queryParam include string Relations to include. Example: author,genres
 *
 * @apiResourceCollection App\Http\Resources\BookResource
 * @apiResourceModel App\Models\Book paginate=15 with=author,genres
 */
public function index(Request $request) { /* ... */ }
```

**Method level** — template for a write endpoint with a Form Request:

```php
/**
 * Create a book
 *
 * @authenticated
 *
 * @apiResource status=201 App\Http\Resources\BookResource
 * @apiResourceModel App\Models\Book with=author,genres
 *
 * @response 422 scenario="Validation failed" {
 *   "error": {
 *     "status": 422,
 *     "message": "The title field is required.",
 *     "details": {"title": ["The title field is required."]}
 *   }
 * }
 */
public function store(StoreBookRequest $request) { /* ... */ }
```

Rules:

- `@group` once at class level, in English, with a stable name (it becomes a section in the docs).
- The first line of the method PHPDoc is the endpoint **title** shown in the sidebar. Keep it short and imperative ("Create a book", not "This method creates a book").
- `@authenticated` / `@unauthenticated` explicitly on **every** method. Never rely on the `auth.default` config: the config must be `'default' => false`.
- `@queryParam` for every query-string parameter the controller reads at runtime. Scribe cannot detect them automatically: if you don't declare them, they won't appear in the docs.
- `@urlParam` is only needed to override the auto-detected name, description, or example.
- Body parameters: when the endpoint uses a Form Request, put descriptions and examples in a `bodyParameters()` method inside the Form Request itself (see the Validation section). For endpoints with inline validation (e.g. auth endpoints), use `@bodyParam` tags in the controller PHPDoc instead.
- Use `@apiResource` / `@apiResourceCollection` + `@apiResourceModel` to let Scribe auto-generate success response examples from a real resource and a factory. Prefer this over hardcoded `@response` blocks for the happy path.
- Use `@response` with a `scenario="..."` attribute for error cases (404, 409, 422) where the shape is known but cannot be inferred from a resource.

### Config

In `config/scribe.php`:

- `'theme' => 'elements'` — use the Stoplight Elements theme; it renders a three-pane layout with a persistent "Try It" console and is significantly more usable for an API reference than the default theme.
- `'auth' => ['default' => false, ...]` — never mark endpoints as authenticated by default. Every method must opt in explicitly via `@authenticated`.

### Readable cURL body in bash examples

Scribe's default Blade template for bash examples wraps the JSON body in double quotes with escaped `\"` on a single line, which is unreadable. Override the template to wrap the body in single quotes with `JSON_PRETTY_PRINT` so the examples are copy-paste friendly.

Create the file `resources/views/vendor/scribe/partials/example-requests/bash.md.blade.php` with the content below. The only change versus the vendor default is on the `--data` line inside the JSON branch:

```blade
@php
    use Knuckles\Scribe\Tools\WritingUtils as u;
    /** @var  Knuckles\Camel\Output\OutputEndpointData $endpoint */
@endphp
```bash
curl --request {{$endpoint->httpMethods[0]}} \
    {{$endpoint->httpMethods[0] == 'GET' ? '--get ' : ''}}"{!! rtrim($baseUrl, '/') !!}/{{ ltrim($endpoint->boundUri, '/') }}@if(count($endpoint->cleanQueryParameters))?{!! u::printQueryParamsAsString($endpoint->cleanQueryParameters) !!}@endif"@if(count($endpoint->headers)) \
@foreach($endpoint->headers as $header => $value)
    --header "{{$header}}: {{ addslashes($value) }}"@if(! ($loop->last) || ($loop->last && count($endpoint->bodyParameters))) \
@endif
@endforeach
@endif
@if($endpoint->hasFiles() || (isset($endpoint->headers['Content-Type']) && $endpoint->headers['Content-Type'] == 'multipart/form-data' && count($endpoint->cleanBodyParameters)))
@foreach($endpoint->cleanBodyParameters as $parameter => $value)
@foreach(u::getParameterNamesAndValuesForFormData($parameter, $value) as $key => $actualValue)
    --form "{!! "$key=".$actualValue !!}"@if(!($loop->parent->last) || count($endpoint->fileParameters))\
@endif
@endforeach
@endforeach
@foreach($endpoint->fileParameters as $parameter => $value)
@foreach(u::getParameterNamesAndValuesForFormData($parameter, $value) as $key => $file)
    --form "{!! "$key=@".$file->path() !!}" @if(!($loop->parent->last))\
@endif
@endforeach
@endforeach
@elseif(count($endpoint->cleanBodyParameters))
@if ($endpoint->headers['Content-Type'] == 'application/x-www-form-urlencoded')
    --data "{!! http_build_query($endpoint->cleanBodyParameters, '', '&') !!}"
@else
{{-- Customized from vendor default: wrap JSON body in single quotes (readable, no \" escapes). Any single quote inside the JSON is escaped with the bash idiom '\'' (close, literal ', reopen). --}}
    --data '{!! str_replace("'", "'\\''", json_encode($endpoint->cleanBodyParameters, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT)) !!}'
@endif
@endif

```
```

The resulting cURL example looks like:

```bash
curl --request POST \
    "http://localhost:8000/api/books" \
    --header "Content-Type: application/json" \
    --data '{
    "title": "The Name of the Rose",
    "isbn": "9780156001311"
}'
```

instead of the default `--data "{\"title\":\"The Name of the Rose\",\"isbn\":\"9780156001311\"}"`.

---

## Security checklist

Before considering a new resource "done", verify:

- [ ] `$fillable` does NOT include `user_id` or any computed/server-controlled field
- [ ] Controller uses `$request->validated()` (never `$request->all()`)
- [ ] If the resource has an owner, `user_id` is set server-side via the user's relationship
- [ ] Policy exists and is registered; `$this->authorize(...)` is called in every controller action
- [ ] Validation includes `exists:...` for every foreign key
- [ ] Controller returns an API Resource (never the model directly)
- [ ] Relations are eager loaded before returning (no N+1)
- [ ] Tests cover both authentication (401) and authorization (403) failure paths

---

## Filtering, sorting, pagination, includes

For collection endpoints, support a consistent query-string API:

- `?page=N` and `?per_page=N` for pagination (clamp `per_page` to a maximum, e.g. 100).
- `?sort=field` and `?sort=-field` for ascending/descending sorting.
- `?include=relation1,relation2` for on-demand eager loading of relations.
- Resource-specific filters (`?language=en`, `?year_from=2020`, `?genre=fantasy`).

Implement these via Builder macros (`withSorting`, `withIncludes`) registered in `AppServiceProvider`, so every resource that needs them can chain the same methods. Validate allowed sort columns and allowed include relations against explicit allowlists; never accept arbitrary column names from the query string.

---

## Laravel version

These conventions target Laravel 13 and PHP 8.3+. When generating code, use modern PHP features (constructor property promotion, readonly properties, typed properties, `match`, enums) and Laravel 13 features (`install:api`, the application builder in `bootstrap/app.php`, `health: '/up'` route, event auto-discovery).
