# isc344 final project
## Brief Description
This project is a fill-in-the-blank activity with a word bank containing draggable elements that allows a user to build their own poem. The poem template is an English translation by myself of the Chinese poem "Fill-in-the-Blanks 填充題" by contemporary Taiwanese poet Hsiao I-Hui 蕭詒徽. Users may store their poem in a database, retrieve their poem (or any poem) from the database, and send their poem to me via LINE (though they probably have no reason to do that).

## Frontend

### Relevant Files
* `fill-in-the-blanks/index.html`: Contains the HTML for the web app.
* `fill-in-the-blanks/index.js`, `fill-in-the-blanks/items.js`: Contains the JavaScript code to set up the fill-in-the-blank interactivity of the web app.
* `fill-in-the-blanks/style.css`, `style.css`: Contains the CSS for the web app.
* `buttons.css`: Contains the CSS for the buttons in the web app.
* `util.js`: Contains JavaScript code to send HTTP requests to server.

### Implementation
* **Drag & Drop:** This app makes use of the [interact.js](https://interactjs.io/) library to enable the drag and drop of elements. The library is included in the HTML file as a script:
  ```html
  <script src="https://unpkg.com/interactjs/dist/interact.min.js"></script>
  ```
* **Snap into Place:** The illusion of draggable elements snapping into place over a blank is achieved by detecting the overlap between a draggable element and a blank space. Upon detection of sufficient overlap, the draggable element is hidden from view:
  * `fill-in-the-blanks/index.js`:
    ```js
    item.classList.add("hidden");
    ```
  * `fill-in-the-blanks/style.css`:
    ```css
    .draggable.hidden {
      visibility: hidden;
    }
    ```
  The blank space is replaced with the text in the draggable element.
  
  Blank spaces are originally initialized by setting the opacity of placeholder texts to zero while maintaining a black underline, as in the CSS below:

  ```css
  span.blank-underline {
    text-decoration: underline;
    text-decoration-color: black;
    text-decoration-thickness: 0.6px;
    text-underline-offset: 1.5px;
    color: rgba(0, 0, 0, 0);
  }
  ```
  After the invisible placeholder text of a "blank" space is replaced by text from a draggable element, the text is set to a visible color:
  ```css
  span.blank-underline.text-added {
    color: rgb(123, 132, 212);
    cursor: pointer;
  }
  ```
  The cursor is also set to `pointer` to indicate that clicking on the text allows the user to "detach" the text from the blank space.
* **Editable Title:** The editable title is achieved by adding the `contenteditable` attribute to a `p` element. Simple logic is added using JavaScript and CSS to enable color change of the text upon editing.
  ```html
  <p contenteditable="true" ... >Title</p>
  ```
* **Store a Poem:** This action is handled by the function `storeAction()` in `util.js`. A `POST` request is made to the `save.php` file on the server with the following data:
  ```js
  {
    action: 'store',
    id: document.getElementById('id').value,
    title: document.getElementById('own-title').innerHTML,
    poem: document.getElementById('poemContent').innerHTML
  };
  ```
* **Retrieve a Poem:** This action is handled by the function `retrieveAction()` in `util.js`. A `POST` request is made to the `save.php` file on the server with the following data (`poemId` is the ID of the poem to be retrieved):
  ```js
  {
    action: 'retrieve',
    id: poemId
  };
  ```
* **Send a Poem:** This action is handled by the function `sendAction()` in `util.js`. A `POST`　request is made to the `save.php` file on the server with the following data:
  ```js
  {
    action: 'notify',
    message: message,
    access_token: accessToken
  }
  ```
  The `message` is the poem to be sent, and the `accessToken` is the access token to a LINE Notify chat connected to my LINE account.


## Backend
### Relevant Files
* `save.php`: Contains PHP code to handle HTTP requests from client.

### Implementation
* **Database:** [MariaDB](https://mariadb.org/) is used to store the poems.
* **Store a Poem:** A `POST` request with a `store` action calls the `store_poem($pdo, $id, $title, $poem)` function in `save.php`. This function first removes certain HTML attributes that are not necessary in a stored poem, including `id`, `title`, `data-line`, and `data-word` (the latter two are customized attributes of my own):
  ```php
  $poem = preg_replace('/<span([^>]*)\bid="[^"]*"(.*?)>/', '<span$1$2>', $poem);
  $poem = preg_replace('/<span([^>]*)\bdata-line="[^"]*"(.*?)>/', '<span$1$2>', $poem);
  $poem = preg_replace('/<span([^>]*)\bdata-word="[^"]*"(.*?)>/', '<span$1$2>', $poem);
  $poem = preg_replace('/<span([^>]*)\btitle="[^"]*"(.*?)>/', '<span$1$2>', $poem);
  ```
  The function then stores the processed HTML into the database. If the given ID already exists in the `poems` database table, the associated poem is updated:
  ```sql
  const UPDATE_SQL = 'UPDATE poems SET title = :title, poem = :poem WHERE id = :id';
  ```
   If not, a new entry is added to the database:
   ```sql
   const INSERT_SQL = 'INSERT INTO poems (id, title, poem) VALUES (:id, :title, :poem)';
   ```
* **Retrieve a Poem:** A `POST` request with a `retrieve` action calls the `retrieve_poem($pdo, $id)` function in `save.php`. This function retrieves a poem from the database:
  ```sql
  const SELECT_SQL = 'SELECT * FROM poems WHERE id = :id';
  ```
* **Send a Poem:** A `POST` request with a `notify` action calls the `send_line_notification($message, $accessToken)` function in `save.php`. This function makes a `POST` request using [cURL](https://www.php.net/manual/en/book.curl.php) to the LINE Notify API at `https://notify-api.line.me/api/notify`:
  ```php
  $url = 'https://notify-api.line.me/api/notify';
  $headers = [
    'Authorization: Bearer ' . $accessToken,
    'Content-Type: application/x-www-form-urlencoded'
  ];

  $postData = http_build_query(['message' => $message]);

  $ch = curl_init($url);
  curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
  curl_setopt($ch, CURLOPT_`POST`, true);
  curl_setopt($ch, CURLOPT_`POST`FIELDS, $postData);
  curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

  $response = curl_exec($ch);
  ```
