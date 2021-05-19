# Debugging Network Requests

## Learning Goals

- Identify where to look when there are issues communicating between client and
  server
- Use the browser developer tools to debug on the frontend
- Use server logging tools and `byebug` to isolate errors on the backend

## Introduction

Now that we're able to make requests using JavaScript to our Rails server, let's
focus on some common issues that arise when making network requests, and learn
techniques to help debug.

To get the backend set up, run:

```sh
bundle install
rails db:migrate db:seed
rails s
```

Then, in a new terminal, run the frontend:

```sh
npm install --prefix client
npm start --prefix client
```

Confirm both applications are both up and running by visiting
[`localhost:4000`](http://localhost:4000) and viewing the list of movies in your
React application.

> Note: This lesson is written using Chrome as our browser of choice for
> debugging. If you're using a different browser, the names and behavior of some
> of the debugging tools may vary.

## Frontend Debugging Tools

To understand the debugging process, let's start building a new feature in our
app, and let our users create new movies. The RESTful way to build a feature
like this would be to allow users to make a `POST` request to `/movies`. In our
React app, there's some code in place for this request already:

```js
// client/src/components/MovieForm.js

function handleSubmit(e) {
  e.preventDefault();
  fetch("/movies", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify(formData),
  })
    .then((r) => r.json())
    .then((data) => {
      console.log(data);
    });
}
```

When the user submits the form, we'll use `fetch` to make a `POST` request to
`/movies` with all the data from the form in the body of the request.

Try it out now, before adding any code to handle this in the backend!

Visit [`localhost:4000/new`](http://localhost:4000/new) to view the form, and
click the Add Movie button to submit it.

What went wrong? How can you tell what went wrong? Where would you look to debug
this error?

From the user's perspective, it appears that nothing happened: we're not seeing
any error messages, but we're also not seeing any indication that our request
went through.

From a **developer's** perspective, we'll need to investigate more closely to see
why this request didn't work.

The first place we can look is in the developer tools, in the **Console** tab.
You'll see an error message that looks like this:

```txt
POST http://localhost:4000/movies 404 (Not Found)
```

As you may have anticipated, since we haven't created this route on the backend
yet, Rails sends a 404 error when we make this request. And any time we use
`fetch` to make a network request that returns a bad status code (400 or 500),
the browser will display an error message about this in the console.

An even better place to look than the Console tab when debugging network request
problems is in the **Network** tab. Open the Network tab, and click the Add
Movie button to submit the form again. You'll see a new request in the Network
tab. Click the request to view more details:

![network 404 request headers](https://raw.githubusercontent.com/learn-co-curriculum/phase-4-debugging-network-requests/master/network-404-headers.png)

Here we can see the 404 status code, as well as some additional information
about the request. Clicking the Preview tab will give more details about the
response the server sent back:

![network 404 response preview](https://raw.githubusercontent.com/learn-co-curriculum/phase-4-debugging-network-requests/master/network-404-preview.png)

Since our Rails server is running in Development mode, we can see some
additional useful information here, such as the exception message that was
thrown.

If you open your terminal where your Rails server is running, you'll see
the same message in the server logs:

```txt
Started POST "/movies" for ::1 at 2021-05-08 10:02:41 -0400

ActionController::RoutingError (No route matches [POST] "/movies"):
```

Now that we know what the problem is, let's start fixing it!

## Debugging the Response

In your Rails app, add a new route:

```rb
# config/routes.rb
Rails.application.routes.draw do
  resources :movies, only: [:index, :create]
end
```

Then, add a controller method:

```rb
class MoviesController < ApplicationController

  def create
  end

  def index
    movies = Movie.all
    render json: movies
  end

end
```

Now, let's try and make the same request from our frontend by submitting the
form again. Our request still isn't working, but now we're no longer getting
that routing error. Instead, our Network tab shows a successful request:

![network 204 no content](https://raw.githubusercontent.com/learn-co-curriculum/phase-4-debugging-network-requests/master/network-204-headers.png)

But our Console tab tells a different story:

```txt
Uncaught (in promise) SyntaxError: Unexpected end of JSON input
  at MovieForm.js:19
```

**Pay careful attention to this error!** This is a very common error message
to see as you're developing API-driven applications. What do you think this
error message indicates?

Well, it's a
[`SyntaxError`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SyntaxError),
which means it's an error that occurs when JavaScript is parsing our code.

It's coming from our `fetch` request as part of the `promise` chain:

```js
fetch("/movies", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify(formData),
})
  .then((r) => r.json())
  .then((data) => {
    console.log(data);
  });
```

The reason for this specific error is that **we expect the server to return a
string of JSON-formatted data**, but the server is not returning any content.

Remember, when we call `r.json()`, we are instructing JavaScript to read the
response (`r`) and parse it as a JSON-formatted string. If the response does not
contain JSON-formatted data, **this code will error out**. We are not sending
**any** data in the response:

```rb
def create
end
```

Keep this in mind when you see this error message! The solution to this problem
is to:

- **return JSON data in the response** from your controller actions, or
- **don't use the `.json()` method on the response** if your controller does not
  respond with JSON data

## Debugging Params

Let's continue working on this feature, and see how we can access the data from
the fetch request in our controller action. Start by adding a `byebug`:

```rb
def create
  byebug
end
```

Then, enter some data in the form, and submit the form again to make another
request. In your server tab, you'll now enter a debugging session:

```txt
    8:   def create
    9:     byebug
=> 10:   end
   11:
   12: end
(byebug)
```

Use this as an opportunity to inspect the request object, in particular looking
at the `params` hash:

```txt
(byebug) params
#<ActionController::Parameters {"title"=>"", "year"=>2021, "length"=>"0",
"director"=>"", ...>
```

You'll see that all the key-value pairs from the `formData` object —
representing the input fields from the form that were sent in the body of the
request — are present in the params hash. Type `c` and hit enter to exit the
`byebug`.

We can use that to create a new `Movie`, and send the movie object back as a
response:

```rb
def create
  movie = Movie.create(movie_params)
  render json: movie, status: :created
end

private

def movie_params
  params.permit(:title, :year, :length, :director, :description, :poster_url, :category, :discount, :female_director)
end
```

With that in place, we can submit the request one more time to finally
create a `Movie`! A good place to look after a successful request is in the
Rails logs to see all the details of the request:

```txt
Started POST "/movies" for ::1 at 2021-05-08 10:36:35 -0400

Processing by MoviesController#create as */* Parameters: {"title"=>"",
"year"=>2021, "length"=>"0", "director"=>"", "description"=>"",
"poster_url"=>"", "category"=>"", "discount"=>false, "female_director"=>false,
"movie"=>{"title"=>"", "year"=>2021, "length"=>"0", "director"=>"",
"description"=>"", "poster_url"=>"", "category"=>"", "discount"=>false,
"female_director"=>false}}

Unpermitted parameter: :movie
  TRANSACTION (0.1ms)  begin transaction
  ↳ app/controllers/movies_controller.rb:9:in `create'

  Movie Create (3.2ms) INSERT INTO "movies" ("title", "year", "length",
  "director", "description", "poster_url", "category", "discount",
  "female_director", "created_at", "updated_at") VALUES (?, ?, ?, ?, ?, ?, ?, ?,
  ?, ?, ?) [["title", ""], ["year", 2021], ["length", 0], ["director", ""],
  ["description", ""], ["poster_url", ""], ["category", ""], ["discount", 0],
  ["female_director", 0], ["created_at", "2021-05-08 14:36:35.970843"],
  ["updated_at", "2021-05-08 14:36:35.970843"]] ↳
  app/controllers/movies_controller.rb:9:in `create'

  TRANSACTION (0.9ms)  commit transaction
  ↳ app/controllers/movies_controller.rb:9:in `create'

Completed 201 Created in 9ms (Views: 0.3ms | ActiveRecord: 4.1ms | Allocations: 2132)
```

You can also view the Network tab to see a preview of the response data:

![network 201 created preview](https://raw.githubusercontent.com/learn-co-curriculum/phase-4-debugging-network-requests/master/network-201-preview.png)

## Debugging Server Errors

Let's introduce one more error on the server to show another common class
of errors we'll need to handle: server-side errors. In the `create` action,
let's imagine we made a typo (as we all do from time to time):

```rb
def create
  # Movi instead of Movie
  movie = Movi.create(movie_params)
  render json: movie
end
```

Submit the form again to initiate another request to this controller action.
You'll see a new error message in the browser console:

```txt
POST http://localhost:4000/movies
  500 (Internal Server Error)
```

Where would you look to debug this error? Well, there are a couple options, but
your best bet any time there is an error **on the server** as indicated with a
500 status code is to look at the **server logs**.

Check your Rails server logs in the terminal, and you'll see:

```txt
Started POST "/movies" for ::1 at 2021-05-08 10:44:56 -0400
   (0.1ms)  SELECT sqlite_version(*)

...

NameError (uninitialized constant MoviesController::Movi):

app/controllers/movies_controller.rb:9:in `create'
```

This gives us a clear place to look for the error. We can see:

- the error itself: `NameError (uninitialized constant MoviesController::Movi)`,
  indicating that we're using a constant `Movi` incorrectly
- the file and line number of the error:
  ` app/controllers/movies_controller.rb:9:in ```create' `

Now, we know exactly where the error is! Sometimes your errors will be more
difficult to find, but utilizing the stack trace and the error message will give
a good indication of where to start looking.

## Conclusion

When you see a `404 Not Found` error:

- Check the Network Tab on the frontend
- Check the Rails Server logs in the backend
- Add a new route to handle the HTTP Verb + Path for this request

When you see an `Unexpected end of JSON input` error:

- Check your `fetch` request
- Check the controller action, and make sure to `render json:`

When you need to see data from the request in your controller:

- Add a `byebug` to your controller method
- Make a request from the browser
- Inspect `params` in the `byebug` session

When you see a `500 Internal Server Error` error:

- Open your Rails server log
- Look for the last request that came through
- Find the error message and stack trace to identify where the error originated
