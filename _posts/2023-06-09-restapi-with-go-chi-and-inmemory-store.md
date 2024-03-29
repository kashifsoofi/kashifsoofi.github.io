---
title:  "REST API with Go, Chi and InMemory Store"
date:   2023-06-09
categories:
  - Go
  - rest
tags:
  - Go
  - rest
  - api
---

## What is REST API?
An API, or application programming interface, is a set of rules that define how applications or devices can connect to and communicate with each other. A REST API is an API that conforms to the design principles of the REST, or representational state transfer architectural style. For this reason, REST APIs are sometimes referred to RESTful APIs.

Focus of this tutorial is to write a REST API with Go.

# Movie Resource
We will be managing a `Movie` resource with the current project. It is not an accurate representation of how you would model a movie resource in an actual system, just a mix of a few basic types and how to handle them in a REST API.

| Field       | Type    |
|-------------|---------|
| ID          | UUID    |
| Title       | String  |
| Director    | String  |
| Director    | String  |
| ReleaseDate | Time    |
| TicketPrice | float64 |

## Project Setup
Create a folder for project, I named it as `movies-api-with-go-chi-and-memory-store` but it usually will be the root of the GitHub repo so you can name it appropriately e.g. `movies-api`.

Execute following command to initialise `go.mod` on terminal  
```shell
go mod init github.com/kashifsoofi/blog-code-samples/movies-api-with-go-chi-and-memory-store
```

Add a new file `main.go` with following content to start with  
```go
package main

func main() {
	println("Hello, World!")
}
```

## Project Structure
I like to add sub-packages to group related functionality together. To that extent I will be adding 3 root-level folders and 1 sub-folder in the `store` folder. Folder structure will be as follows (not showing files).

```
.
└── movies-api-with-go-chi-and-memory-store/
    ├── api - this will contain rest routes, handlers etc.
    ├── config - this will contain anything related to service configuration
    └── store/ - this will contain store interface
```

I have only 2 resources, `health` and `movies`, however if you are serving more resources in a single rest service, feel free to add a folder per resource under `api`. Same for the `store` if you are adding multiple `stores` e.g. `Postgres` and a `Redis` cache to check before hitting `Postgres` then feel free to add specific folders for each store.

## Configuration
Add a folder named `config` and a file named `config.go`. I like to keep all application configuration in a single place and will be using excellent `envconfig` package to load the configuration, also setting some default values for options. This package allows us to load application configuration from Environment Variables, same thing can be done with standard Go packages but in my opinion this package provides nice abstraction without losing readability.
```go
package config

import (
	"time"

	"github.com/kelseyhightower/envconfig"
)

const envPrefix = ""

type Configuration struct {
	HTTPServer
}

type HTTPServer struct {
	IdleTimeout  time.Duration `envconfig:"HTTP_SERVER_IDLE_TIMEOUT" default:"60s"`
	Port         int           `envconfig:"PORT" default:"8080"`
	ReadTimeout  time.Duration `envconfig:"HTTP_SERVER_READ_TIMEOUT" default:"1s"`
	WriteTimeout time.Duration `envconfig:"HTTP_SERVER_WRITE_TIMEOUT" default:"2s"`
}

func Load() (Configuration, error) {
	var cfg Configuration
	err := envconfig.Process(envPrefix, &cfg)
	if err != nil {
		return cfg, err
	}

	return cfg, nil
}
```
This would result in an error that can be resolved by executing following on terminal.
```shell
go mod tidy
```

You can improve on it by converting `Configuration` to an `interface` and then adding a configuration to each of sub-packages e.g. `api`, `store` etc.

Configuration can be updated using environment variables e.g. executing following on terminal would start the server on port 5000 after we update `main.go` to start the server.
```shell
PORT=5000 go run main.go
```

## Movie Store Interface
Add a new folder named `store` and a file named `movie_store.go`. We will add an interface for our movie store and supporting structs.
```go
package store

import (
	"time"

	"github.com/google/uuid"
)

type Movie struct {
	ID          uuid.UUID
	Title       string
	Director    string
	ReleaseDate time.Time
	TicketPrice float64
	CreatedAt   time.Time
	UpdatedAt   time.Time
}

type CreateMovieParams struct {
	ID          uuid.UUID
	Title       string
	Director    string
	ReleaseDate time.Time
	TicketPrice float64
}

type UpdateMovieParams struct {
	Title       string
	Director    string
	ReleaseDate time.Time
	TicketPrice float64
}

type Interface interface {
	GetAll() ([]Movie, error)
	GetByID(id uuid.UUID) (Movie, error)
	Create(createMovieParams CreateMovieParams) error
	Update(id uuid.UUID, updateMovieParams UpdateMovieParams) error
	Delete(id uuid.UUID) error
}
```

Also add a file for custom application errors named `errors.go`, these make clients of our store package agnostic of storage technology used, our storage implementation would translate any native errors to our business errors before returning to clients. 
```go
package store

import (
	"fmt"

	"github.com/google/uuid"
)

type DuplicateKeyError struct {
	ID uuid.UUID
}

func (e *DuplicateKeyError) Error() string {
	return fmt.Sprintf("duplicate movie id: %v", e.ID)
}

type RecordNotFoundError struct{}

func (e *RecordNotFoundError) Error() string {
	return "record not found"
}
```

## MemoryMoviesStore
Add a new file named `memory_movies_store.go` in `store` folder. Add a struct `MemoryMoviesStore` with a map field to store movies in memory. Also add a `RWMutex` field to avoid concurrent read/write access to movies field.

We will implement all methods defined for `store.Interface` to add/remove movies to map field of the `MemoryMoviesStore` struct. For reading we lock the collection for reading, read the result and release the lock using `defer`. For writing we acquire a write lock instead of a read lock.

```go
package store

import (
	"sync"
	"time"

	"github.com/google/uuid"
)

type MemoryMoviesStore struct {
	movies map[uuid.UUID]Movie
	mu     sync.RWMutex
}

func NewMemoryMoviesStore() *MemoryMoviesStore {
	return &MemoryMoviesStore{
		movies: map[uuid.UUID]Movie{},
	}
}

func (s *MemoryMoviesStore) GetAll() ([]Movie, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	var movies []Movie
	for _, m := range s.movies {
		movies = append(movies, m)
	}
	return movies, nil
}

func (s *MemoryMoviesStore) GetByID(id uuid.UUID) (Movie, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	m, ok := s.movies[id]
	if !ok {
		return Movie{}, &RecordNotFoundError{}
	}

	return m, nil
}

func (s *MemoryMoviesStore) Create(createMovieParams CreateMovieParams) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	if _, ok := s.movies[createMovieParams.ID]; ok {
		return &DuplicateKeyError{ID: createMovieParams.ID}
	}

	movie := Movie{
		ID:          createMovieParams.ID,
		Title:       createMovieParams.Title,
		Director:    createMovieParams.Director,
		ReleaseDate: createMovieParams.ReleaseDate,
		TicketPrice: createMovieParams.TicketPrice,
		CreatedAt:   time.Now().UTC(),
		UpdatedAt:   time.Now().UTC(),
	}

	s.movies[movie.ID] = movie
	return nil
}

func (s *MemoryMoviesStore) Update(id uuid.UUID, updateMovieParams UpdateMovieParams) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	m, ok := s.movies[id]
	if !ok {
		return &RecordNotFoundError{}
	}

	m.Title = updateMovieParams.Title
	m.Director = updateMovieParams.Director
	m.ReleaseDate = updateMovieParams.ReleaseDate
	m.TicketPrice = updateMovieParams.TicketPrice
	m.UpdatedAt = time.Now().UTC()

	s.movies[id] = m
	return nil
}

func (s *MemoryMoviesStore) Delete(id uuid.UUID) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	delete(s.movies, id)
	return nil
}
```

## REST Server
Add a new folder to add all REST api server related files. Let's start by adding `server.go` file and add a struct to represent REST server. This struct would have an instance of configuration required to run server, routes and all the dependencies. Also add method to start the server.
For routes we would use excellent `chi` router, that is a ligtweight, idomatic and composable router for building HTTP services.

In start method, we will construct an instance of `Server` provided by standard `net/http` package, providing `chi mux` we setup in `NewServer` method. We will then setup a method for graceful shutdown and call `ListenAndServe` to start our REST server.
```go
package api

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"

	"github.com/kashifsoofi/blog-code-samples/movies-api-with-go-chi-and-memory-store/config"
	"github.com/kashifsoofi/blog-code-samples/movies-api-with-go-chi-and-memory-store/store"

	"github.com/go-chi/chi/v5"
)

type Server struct {
	cfg    config.HTTPServer
	store  store.Interface
	router *chi.Mux
}

func NewServer(cfg config.HTTPServer, store store.Interface) *Server {
	srv := &Server{
		cfg:    cfg,
		store:  store,
		router: chi.NewRouter(),
	}

	srv.routes()

	return srv
}

func (s *Server) Start(ctx context.Context) {
	server := http.Server{
		Addr:         fmt.Sprintf(":%d", s.cfg.Port),
		Handler:      s.router,
		IdleTimeout:  s.cfg.IdleTimeout,
		ReadTimeout:  s.cfg.ReadTimeout,
		WriteTimeout: s.cfg.WriteTimeout,
	}

	shutdownComplete := handleShutdown(func() {
		if err := server.Shutdown(ctx); err != nil {
			log.Printf("server.Shutdown failed: %v\n", err)
		}
	})

	if err := server.ListenAndServe(); err == http.ErrServerClosed {
		<-shutdownComplete
	} else {
		log.Printf("http.ListenAndServe failed: %v\n", err)
	}

	log.Println("Shutdown gracefully")
}

func handleShutdown(onShutdownSignal func()) <-chan struct{} {
	shutdown := make(chan struct{})

	go func() {
		shutdownSignal := make(chan os.Signal, 1)
		signal.Notify(shutdownSignal, os.Interrupt, syscall.SIGTERM)

		<-shutdownSignal

		onShutdownSignal()
		close(shutdown)
	}()

	return shutdown
}
```
## Custom API Errors
We would define any custom errors that are returned by our REST server in `errors.go` file under `api` folder. I have gone ahead and added all the errors I need to return from this service in the file. But practically we would start with the most common ones and then add any new when the need arise.
```go
package api

import (
	"net/http"

	"github.com/go-chi/render"
)

type ErrResponse struct {
	Err            error `json:"-"` // low-level runtime error
	HTTPStatusCode int   `json:"-"` // http response status code

	StatusText string `json:"status"`          // user-level status message
	AppCode    int64  `json:"code,omitempty"`  // application-specific error code
	ErrorText  string `json:"error,omitempty"` // application-level error message, for debugging
}

func (e *ErrResponse) Render(w http.ResponseWriter, r *http.Request) error {
	render.Status(r, e.HTTPStatusCode)
	return nil
}

var (
	ErrNotFound            = &ErrResponse{HTTPStatusCode: 404, StatusText: "Resource not found."}
	ErrBadRequest          = &ErrResponse{HTTPStatusCode: 400, StatusText: "Bad request"}
	ErrInternalServerError = &ErrResponse{HTTPStatusCode: 500, StatusText: "Internal Server Error"}
)

func ErrConflict(err error) render.Renderer {
	return &ErrResponse{
		Err:            err,
		HTTPStatusCode: 409,
		StatusText:     "Duplicate Key",
		ErrorText:      err.Error(),
	}
}
```
## Routes
I like the keep all the routes served by a service in a single place and single file named `routes.go`. Its easier to remember and eases the cognitive overload.

`routes` method hangs off our `Server` struct, defines all the endpoints on the `router` field. I have defined a `/health` endpoint that would return current health status of this service. Then added a subrouter group for movies. This can help us having middlewared applied only for `/api/movies` routes e.g. authentication, request logging.
```go
package api

import (
	"github.com/go-chi/chi/v5"
	"github.com/go-chi/render"
)

func (s *Server) routes() {
	s.router.Use(render.SetContentType(render.ContentTypeJSON))

	s.router.Get("/health", s.handleGetHealth)

	s.router.Route("/api/movies", func(r chi.Router) {
		r.Get("/", s.handleListMovies)
		r.Post("/", s.handleCreateMovie)
		r.Route("/{id}", func(r chi.Router) {
			r.Get("/", s.handleGetMovie)
			r.Put("/", s.handleUpdateMovie)
			r.Delete("/", s.handleDeleteMovie)
		})
	})
}
```
Please note all handlers hang off the `Server` struct, this helps to access required dependencies in each of the handlers. If there are multiple resources in a service, it might make sense to add separate `structs` per resource containing only dependencies required by that resource.

## Health Endpoint Handler
I have added a separate file for `health` resource. It has a handler for a single endpoint, a struct that we would send as response and implementation of `Renderer` interface for our response struct.
```go
package api

import (
	"net/http"

	"github.com/go-chi/render"
)

type healthResponse struct {
	OK bool `json:"ok"`
}

func (hr healthResponse) Render(w http.ResponseWriter, r *http.Request) error {
	return nil
}

func (s *Server) handleGetHealth(w http.ResponseWriter, r *http.Request) {
	health := healthResponse{OK: true}
	render.Render(w, r, health)
}
```

## Movies Endpoints Handlers

### Get Movie By ID
Let's start by adding a struct we would use to return a `Movie` to the caller of our REST service and also implement `Renderer` interface so that we can use `Render` method to return data.
```go
type movieResponse struct {
	ID          uuid.UUID `json:"id"`
	Title       string    `json:"title"`
	Director    string    `json:"director"`
	ReleaseDate time.Time `json:"release_date"`
	TicketPrice float64   `json:"ticket_price"`
	CreatedAt   time.Time `json:"created_at"`
	UpdatedAt   time.Time `json:"updated_at"`
}

func NewMovieResponse(m store.Movie) movieResponse {
	return movieResponse{
		ID:          m.ID,
		Title:       m.Title,
		Director:    m.Director,
		ReleaseDate: m.ReleaseDate,
		TicketPrice: m.TicketPrice,
		CreatedAt:   m.CreatedAt,
		UpdatedAt:   m.UpdatedAt,
	}
}

func (hr movieResponse) Render(w http.ResponseWriter, r *http.Request) error {
	return nil
}
```
I like to keep `structs` closer to the method/package using those. It does lead to some code duplication e.g. in this case `movieResponse` is quite similar to `Movie` struct defind in `store/movies_store.go` file but this allows rest package to be not completely dependent on `store` package, we can have different tags e.g. db specific tags in `store` struct but not in `movieResponse` struct.

Now comes the handler, we receive a `ResponseWriter` and a `Request`, we extract `id` parameter from path using `URLParam` method, if parsing fails we rendera `BadRequest`.

Then we proceed to get `movie` from `store` if a record is not found in store with given `id` we render `NotFound`, if the error returned is not the one we defined in our store package then we render an `InternalServerError`, we can add more custom/known errors to store and translate to apprpriate HTTP Status Codes depending on the use case.

If everything works then we convert the `store.Movie` to `movieResponse` and render the result. Result would be returned to the caller as `json` response body.
```go
func (s *Server) handleGetMovie(w http.ResponseWriter, r *http.Request) {
	idParam := chi.URLParam(r, "id")
	id, err := uuid.Parse(idParam)
	if err != nil {
		render.Render(w, r, ErrBadRequest)
		return
	}

	movie, err := s.store.GetByID(id)
	if err != nil {
		var rnfErr *store.RecordNotFoundError
		if errors.As(err, &rnfErr) {
			render.Render(w, r, ErrNotFound)
		} else {
			render.Render(w, r, ErrInternalServerError)
		}
		return
	}

	mr := NewMovieResponse(movie)
	render.Render(w, r, mr)
}
```

### Get All/List Movies
For response we would use the same `movieResponse` struct we defined for `Get By ID`, we would just add a new method to create an array/slice of `Renderer`
```go
func (s *Server) handleListMovies(w http.ResponseWriter, r *http.Request) {
	movies, err := s.store.GetAll()
	if err != nil {
		render.Render(w, r, ErrInternalServerError)
		return
	}

	render.RenderList(w, r, NewMovieListResponse(movies))
}
```

And the handler method is quite simple, we call `GetAll`, if error return `InternalServerError` else return list of movies.
```go
func (s *Server) handleListMovies() http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		movies, err := s.store.GetAll()
		if err != nil {
			render.Render(w, r, ErrInternalServerError)
			return
		}

		render.RenderList(w, r, NewMovieListResponse(movies))
	}
}
```

### Create Movie
Same as get, we would start by adding a new struct to receive parameters required to create a new movie. But instead of implmenting `Renderer` we would implement `Binder` interface, in `Bind` method custom mapping can be done if required e.g. adding meta data or setting a `CreatedBy` fields from `JWT` token.

Please note we don't have `CreatedAt` and `UpdatedAt` in this struct.
```go
type CreateMovieRequest struct {
	ID          string    `json:"id"`
	Title       string    `json:"title"`
	Director    string    `json:"director"`
	ReleaseDate time.Time `json:"release_date"`
	TicketPrice float64   `json:"ticket_price"`
}

func (mr *createMovieRequest) Bind(r *http.Request) error {
	return nil
}
```

In handler we bind request body to our struct, if `Bind` is successful then convert it to `CreateMovieParams` struct expected by `store.Create` method and call `Create` method to add movie to data store. If there is a duplicate key error we return `409 Conflict` for unknown errors we return `500 InternalServerError` and if all is successful we are returning `200 OK`.
```go
func (s *Server) handleCreateMovie(w http.ResponseWriter, r *http.Request) {
	data := &CreateMovieRequest{}
	if err := render.Bind(r, data); err != nil {
		render.Render(w, r, ErrBadRequest)
		return
	}

	createMovieParams := store.CreateMovieParams{
		ID:          uuid.MustParse(data.ID),
		Title:       data.Title,
		Director:    data.Director,
		ReleaseDate: data.ReleaseDate,
		TicketPrice: data.TicketPrice,
	}
	err := s.store.Create(createMovieParams)
	if err != nil {
		var dupKeyErr *store.DuplicateKeyError
		if errors.As(err, &dupKeyErr) {
			render.Render(w, r, ErrConflict(err))
		} else {
			render.Render(w, r, ErrInternalServerError)
		}
		return
	}

	w.WriteHeader(200)
	w.Write(nil)
}
```

### Update Movie
Same as `Create Movie` above, we introduced a new struct `updateMovieRequest` to receive parameters required to update movie and implemeted `Binder` interface for the struct.
```go
type updateMovieRequest struct {
	Title       string    `json:"title"`
	Director    string    `json:"director"`
	ReleaseDate time.Time `json:"release_date"`
	TicketPrice float64   `json:"ticket_price"`
}

func (mr *updateMovieRequest) Bind(r *http.Request) error {
	return nil
}
```

In hander we read the `id` from path, then we bind the struct from request body. If no errors then we convert the request to `store.UpdateMovieParams` and call `Update` method of store to update movie. We return `200 OK` if upate is successful.
```go
func (s *Server) handleUpdateMovie(w http.ResponseWriter, r *http.Request) {
	idParam := chi.URLParam(r, "id")
	id, err := uuid.Parse(idParam)
	if err != nil {
		render.Render(w, r, ErrBadRequest)
		return
	}

	data := &updateMovieRequest{}
	if err := render.Bind(r, data); err != nil {
		render.Render(w, r, ErrBadRequest)
		return
	}

	updateMovieParams := store.UpdateMovieParams{
		Title:       data.Title,
		Director:    data.Director,
		ReleaseDate: data.ReleaseDate,
		TicketPrice: data.TicketPrice,
	}
	err = s.store.Update(id, updateMovieParams)
	if err != nil {
		var rnfErr *store.RecordNotFoundError
		if errors.As(err, &rnfErr) {
			render.Render(w, r, ErrNotFound)
		} else {
			render.Render(w, r, ErrInternalServerError)
		}
		return
	}

	w.WriteHeader(200)
	w.Write(nil)
}
```

### Delete Movie
This probably is the simplest handler as it does not need any `Renderer` or `Binder`, we simply get `id` from the path, and call `Delete` method of store to delete the resource. If delete is successful we return `200 OK`.
```go
func (s *Server) handleDeleteMovie(w http.ResponseWriter, r *http.Request) {
	idParam := chi.URLParam(r, "id")
	id, err := uuid.Parse(idParam)
	if err != nil {
		render.Render(w, r, ErrBadRequest)
		return
	}

	err = s.store.Delete(id)
	if err != nil {
		var rnfErr *store.RecordNotFoundError
		if errors.As(err, &rnfErr) {
			render.Render(w, r, ErrNotFound)
		} else {
			render.Render(w, r, ErrInternalServerError)
		}
		return
	}

	w.WriteHeader(200)
	w.Write(nil)
}
```

## Start Server
Now everything is setup, its time to update `main` method. Start by laoding the configuration, then create an instance of the `MemoryMoviesStore`, here we can also instantiate any other dependencies our server is dependent on. Next step is to create an instance of `api.Server` struct and call the `Start` method to start the server. Server would start listening on the configured port and you can invoke endpoints using `curl` or `Postman`.
```go
package main

import (
	"context"
	"log"

	"github.com/kashifsoofi/blog-code-samples/movies-api-with-go-chi-and-memory-store/api"
	"github.com/kashifsoofi/blog-code-samples/movies-api-with-go-chi-and-memory-store/config"
	"github.com/kashifsoofi/blog-code-samples/movies-api-with-go-chi-and-memory-store/store"
)

func main() {
	ctx := context.Background()
	cfg, err := config.Load()
	if err != nil {
		log.Fatal(err)
	}

	store := store.NewMemoryMoviesStore()
	server := api.NewServer(cfg.HTTPServer, store)
	server.Start(ctx)
}
```

## Testing
I am going to list steps to manually test the api endpoints, as we don't have `Swagger UI` or any other UI to interact with this, `Postman` can be used to test the endpoints as well.
* Start Server executing following
```shell
go run main.go
```
Execute following tests in order, remember to update the port if you are running on a different port than 8080.

> **_NOTE:_**  I have not added `created_at` and `updated_at` fields in responses below.

### Tests
#### Get All returns empty list
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies"
```
##### Expected Response
```json
[]
```
#### Get By ID should return Not Found
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies/1"
```
##### Expected Response
```json
[]
```
#### Get By ID should return Not Found
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies/1"
```
##### Expected Response
```json
[]
```
#### Get By ID should return Not Found
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies/1"
```
##### Expected Response
```json
[]
```
#### Get By ID with invalid id
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies/1"
```
##### Expected Response
```json
{"status":"Bad request"}
```
#### Get by ID with non-existent record
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies/98268a96-a6ac-444f-852a-c6472129aa22"
```
##### Expected Response
```json
{"status":"Resource not found."}
```
#### Create Movie
##### Request
```shell
curl --request POST --data '{ "id": "98268a96-a6ac-444f-852a-c6472129aa22", "title": "Star Wars: Episode I – The Phantom Menace", "director": "George Lucas", "release_date": "1999-05-16T01:01:01.00Z", "ticket_price": 10.70 }' --url "http://localhost:8080/api/movies"
```
##### Expected Response
```json
```
#### Create Movie with existing ID
##### Request
```shell
curl --request POST --data '{ "id": "98268a96-a6ac-444f-852a-c6472129aa22", "title": "Star Wars: Episode I – The Phantom Menace", "director": "George Lucas", "release_date": "1999-05-16T01:01:01.00Z", "ticket_price": 10.70 }' --url "http://localhost:8080/api/movies"
```
##### Expected Response
```json
{"status":"Duplicate Key","error":"duplicate movie id: 98268a96-a6ac-444f-852a-c6472129aa22"}
```
#### Get ALL Movies
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies"
```
##### Expected Response
```json
[{"id":"98268a96-a6ac-444f-852a-c6472129aa22","title":"Star Wars: Episode I – The Phantom Menace","director":"George Lucas","release_date":"1999-05-16T01:01:01Z","ticket_price":10.7}]
```
#### Get Movie By ID
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies/98268a96-a6ac-444f-852a-c6472129aa22"
```
##### Expected Response
```json
{"id":"98268a96-a6ac-444f-852a-c6472129aa22","title":"Star Wars: Episode I – The Phantom Menace","director":"George Lucas","release_date":"1999-05-16T01:01:01Z","ticket_price":10.7}
```
#### Update Movie
##### Request
```shell
curl --request PUT --data '{ "title": "Star Wars: Episode I – The Phantom Menace", "director": "George Lucas", "release_date": "1999-05-16T01:01:01.00Z", "ticket_price": 20.70 }' --url "http://localhost:8080/api/movies/98268a96-a6ac-444f-852a-c6472129aa22"
```
##### Expected Response
```json
```
#### Get Movie by ID - get updated record
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies/98268a96-a6ac-444f-852a-c6472129aa22"
```
##### Expected Response
```json
{"id":"98268a96-a6ac-444f-852a-c6472129aa22","title":"Star Wars: Episode I – The Phantom Menace","director":"George Lucas","release_date":"1999-05-16T01:01:01Z","ticket_price":20.7}
```
#### Delete Movie
##### Request
```shell
curl --request DELETE --url "http://localhost:8080/api/movies/98268a96-a6ac-444f-852a-c6472129aa22"
```
##### Expected Response
```json
```
#### Get Movie By ID - deleted record
##### Request
```shell
curl --request GET --url "http://localhost:8080/api/movies/98268a96-a6ac-444f-852a-c6472129aa22"
```
##### Expected Response
```json
{"status":"Resource not found."}
```

## Source
Source code for the demo application is hosted on GitHub in [blog-code-samples](https://github.com/kashifsoofi/blog-code-samples/tree/main/movies-api-with-go-chi-and-memory-store) repository.

## References
In no particular order
* [What is a REST API?](https://www.ibm.com/topics/rest-apis)
* [envconfig](https://github.com/kelseyhightower/envconfig)
* [chi](https://github.com/go-chi/chi)
* And many more