# EasySave Frontend - Detailed Code Documentation

## Table of Contents
1. [JavaScript Files](#javascript-files)
   - [login.js](#loginjs)
   - [register.js](#registerjs)
   - [main.js](#mainjs)
2. [HTML Files](#html-files)
   - [login.html](#loginhtml)
   - [register.html](#registerhtml)
   - [main.html](#mainhtml)
3. [CSS Files](#css-files)
   - [login.css](#logincss)
   - [register.css](#registercss)
   - [main.css](#maincss)

---

## JavaScript Files

### login.js

**Purpose**: Handles user authentication, login form submission, and provides interactive mouse-tracking animation for the login box.

#### Global Scope Functions

##### `getCookie(cname)`
Retrieves a cookie value by name from the browser's document.cookie.

**Parameters:**
- `cname` (string): The name of the cookie to retrieve

**Returns:**
- (string): The cookie value if found, empty string otherwise

**Algorithm:**
1. Creates search string with cookie name + "="
2. Decodes the entire cookie string
3. Splits cookies by semicolon
4. Iterates through each cookie
5. Trims whitespace and checks if it starts with the target name
6. Returns the value portion if found

**Usage Example:**
```javascript
const username = getCookie("username");
const auth = getCookie("auth");
```

#### Document Ready Handler

The main functionality is wrapped in a `DOMContentLoaded` event listener to ensure the DOM is fully loaded before executing.

##### Interactive Box Animation System

**State Object:**
```javascript
const state = {
    mouseX: window.innerWidth / 2,   // Current mouse X position
    mouseY: window.innerHeight / 2,   // Current mouse Y position
    tx: 0,                            // Current translation X
    ty: 0,                            // Current translation Y
    targetX: 0,                       // Target translation X
    targetY: 0                        // Target translation Y
};
```

**Constants:**
- `followStrength`: 0.05 (5% of distance between mouse and box center)
- `maxOffset`: Calculated as 10% of the smaller viewport dimension (initially 6% after resize)

##### `lerp(a, b, t)`
Linear interpolation function for smooth transitions.

**Parameters:**
- `a` (number): Start value
- `b` (number): End value
- `t` (number): Interpolation factor (0-1)

**Returns:**
- (number): Interpolated value between a and b

**Formula:** `a + (b - a) * t`

##### `updateTarget()`
Calculates the target translation values based on mouse position.

**Algorithm:**
1. Gets the bounding rectangle of the login box
2. Calculates the center point of the box (cx, cy)
3. Calculates the distance from mouse to box center (dx, dy)
4. Multiplies by followStrength to get initial target
5. Calculates the magnitude of the target vector
6. Clamps to maxOffset if magnitude exceeds it
7. Updates state.targetX and state.targetY

**Purpose:** Creates a "following" effect where the box moves towards the mouse cursor but is limited by maxOffset to prevent excessive movement.

##### `animate()`
Animation loop that smoothly transitions the box position.

**Process:**
1. Calls updateTarget() to get new target position
2. Applies lerp to smoothly approach target (smoothing factor: 0.15)
3. Updates CSS custom properties --tx and --ty
4. Requests next animation frame for continuous animation

**CSS Integration:**
The function sets CSS variables that are used in the transform property:
```css
transform: translate(var(--tx, 0), var(--ty, 0)) scale(1);
```

##### Event Listeners

**mousemove:**
```javascript
window.addEventListener("mousemove", (e) => {
    state.mouseX = e.clientX;
    state.mouseY = e.clientY;
}, { passive: true });
```
- Updates mouse position in state
- Uses passive: true for better scrolling performance

**mouseleave:**
```javascript
window.addEventListener("mouseleave", () => {
    state.mouseX = window.innerWidth / 2;
    state.mouseY = window.innerHeight / 2;
});
```
- Resets mouse position to viewport center when cursor leaves window
- Causes box to smoothly return to center position

**resize:**
```javascript
window.addEventListener("resize", () => {
    maxOffset = Math.min(window.innerWidth, window.innerHeight) * 0.06;
});
```
- Recalculates maxOffset when window is resized
- Maintains proportional movement limit

##### Login Form Submission

**Form Validation:**
- HTML5 required attributes ensure username and password are filled
- Form data is extracted from form.username.value and form.password.value

**API Request:**
```javascript
const xhr = new XMLHttpRequest();
xhr.open("GET", "http://63.179.18.244/api/login?username=" + data.username + "&password=" + data.password, true);
xhr.responseType = "json";
```

**Request Configuration:**
- Method: GET
- Endpoint: `/api/login`
- Parameters: username and password as query strings
- Response Type: JSON

**Response Handling:**

**Success (status 200-299):**
1. Extracts accessKey from response
2. Creates username cookie with 7-day expiration
3. Creates auth cookie with access key and 7-day expiration
4. Redirects to /main page

**Unauthorized (status 401):**
- Displays alert: "Login unsuccessful. Please check username and password."

**Other Errors:**
- Logs error to console
- Displays generic "Login failed" alert

**Cookie Format:**
```javascript
document.cookie = [
    "username=" + encodeURIComponent(data.username),
    "Path=/",
    "Max-Age=" + (60 * 60 * 24 * 7),  // 7 days in seconds
    "SameSite=Lax"
].join("; ");
```

**Auto-redirect Check:**
```javascript
if (getCookie("auth")) {
    window.location.href = "/main";
}
```
- If user is already authenticated, immediately redirects to main page
- Prevents logged-in users from seeing login form

---

### register.js

**Purpose**: Manages user registration, form submission, and provides the same interactive animation effects as login.js.

**Differences from login.js:**
1. Selects `.register-box` instead of `.login-box`
2. Form submission uses POST method instead of GET
3. Includes email field in form data
4. Different API endpoint: `/api/create_user`
5. On success, redirects to `/login` instead of `/main`
6. No cookie creation (user must log in after registration)

#### Registration Form Submission

**Form Fields:**
```javascript
const data = {
    username: form.username.value,
    email: form.email.value,
    password: form.password.value
};
```

**API Request:**
```javascript
xhr.open("POST", "http://63.179.18.244/api/create_user?username=" + data.username + "&password=" + data.password + "&email=" + data.email, true);
```

**Response Handling:**

**Success (status 200-299):**
- Redirects to `/login` page for user to authenticate

**Failure:**
- Displays alert with error details: `xhr.response.detail`
- Logs error to console for debugging

**No Auto-redirect Check:**
- Register page doesn't check for existing authentication
- Users can access registration even if logged in

---

### main.js

**Purpose**: Core application logic for the database management interface. Handles authentication verification, data fetching, CRUD operations, and dynamic UI updates.

#### Global Variables

```javascript
const username = getCookie("username");  // Current authenticated username
const auth = getCookie("auth");          // Access key token
var blocks = [];                         // Array of data blocks (currently unused)
var baseHeight = 30;                     // Base height for textareas in pixels
var emptyRowActive = false;              // Flag to prevent multiple empty rows
```

#### Document Ready Handler

**Authentication Check:**
```javascript
if (!getCookie("auth")) {
    window.location.href = "/login";
}
```
- Verifies user is authenticated
- Redirects to login page if no auth cookie found

**Username Display:**
```javascript
document.getElementById("username").innerText = username
```
- Displays current username in the header

**Refresh Icon Handler:**
```javascript
document.getElementById("refreshIcon").addEventListener("click", async function(e) {
    e.target.classList.add("spin")     // Add spinning animation
    refreshTable()                      // Fetch and render data
    await sleep(1000)                   // Wait 1 second
    e.target.classList.remove("spin")  // Remove spinning animation
});
```

**Logout Icon Handler:**
```javascript
document.getElementById("logoutIcon").addEventListener("click", function(e) {
    // Clear cookies by setting expiration to past date
    document.cookie = "auth=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;";
    document.cookie = "username=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;";
    window.location.href = "/login";
});
```

**Initial Data Load:**
```javascript
refreshTable()
```
- Immediately loads data when page loads

#### Utility Functions

##### `getCookie(cname)`
Identical implementation to login.js - retrieves cookie value by name.

##### `sendRequest(method, endpoint, data, callback)`
Generic API request handler with authentication.

**Parameters:**
- `method` (string): HTTP method (GET, POST, PATCH)
- `endpoint` (string): API endpoint path (e.g., "get_blocks")
- `data` (object): Key-value pairs for query parameters
- `callback` (function): Callback with signature (error, response)

**Process:**
1. Constructs full URL: `http://63.179.18.244/api/` + endpoint
2. Iterates through data object to build query string
3. URL encodes parameter values
4. Appends parameters with ? or & as appropriate
5. Creates XMLHttpRequest with specified method
6. Sets headers:
   - Content-Type: application/json;charset=UTF-8
   - RequesterUsername: current username
   - RequesterAccessKey: auth token
7. Sets responseType to "json"
8. Waits for readyState 4 (complete)
9. Calls callback with (null, response) on success
10. Calls callback with (Error, null) on failure

**Example Usage:**
```javascript
sendRequest("GET", "get_blocks", {"extendedIdentifier": ""}, function(error, response) {
    if (error == null) {
        // Handle success
    } else {
        // Handle error
    }
});
```

##### `sleep(ms)`
Returns a Promise that resolves after specified milliseconds.

**Parameters:**
- `ms` (number): Milliseconds to sleep

**Returns:**
- Promise that resolves after ms milliseconds

**Usage:**
```javascript
await sleep(1000);  // Wait 1 second
```

#### Core Functions

##### `refreshTable()`
Fetches all data blocks from the server and renders them in the table.

**Process Overview:**
1. Capture base textarea height (for auto-resize calculations)
2. Reset emptyRowActive flag
3. Clear existing table content
4. Add column headers from template
5. Fetch data blocks from API
6. Render each block as a table row
7. Add "new entry" row at the bottom
8. Setup event listeners for all textareas

**Detailed Steps:**

**1. Capture Base Height:**
```javascript
try {
    baseHeight = document.getElementsByTagName("textarea")[0].scrollHeight;
} catch (error) {
    // Silently ignore if no textareas exist yet
}
```

**2. Clear and Reset Table:**
```javascript
emptyRowActive = false
table = document.getElementsByClassName("main-table")[0]
table.innerText = ""
table.appendChild(document.getElementById("titlesTemplate").content.cloneNode(true));
```

**3. Fetch Data:**
```javascript
sendRequest("GET", "get_blocks", {"extendedIdentifier": ""}, function(error, response) {
    if (response != null) {
        response.blockList.forEach(function(value) {
            // Process each block
        });
    }
});
```

**4. Render Each Block:**

For each block in response.blockList:

a. Extract identifier (trimmed of first two segments):
```javascript
trimmedIdentifier = value.identifier.split(".").slice(2).join(".")
```

b. Create table row and cells:
```javascript
newTableRow = document.createElement("tr")
```

c. **First Column (Identifier):**
```javascript
tableEntryOne = document.createElement("td")
newInput = document.createElement("input")
newInput.type = "text"
newInput.value = trimmedIdentifier
newInput.disabled = true  // Read-only for existing blocks
```

d. **Second Column (Value):**
```javascript
tableEntryTwo = document.createElement("td")
textArea = document.createElement("textarea")
textArea.value = newValue

// Auto-resize on input
textArea.addEventListener("input", () => {
    textArea.style.height = baseHeight - 6 + "px";        // Reset
    textArea.style.height = textArea.scrollHeight - 6 + "px"  // Expand
});
```

e. **Third Column (Save Icon):**
```javascript
saveIcon = document.createElement("i")
saveIcon.classList = "fa-solid fa-floppy-disk fa-xl save-icon"

// Click handler
saveIcon.addEventListener("click", function(e) {
    identifier = e.target.parentElement.parentElement.children[0].children[0].value
    value = e.target.parentElement.parentElement.children[1].children[0].value
    saveValue(e.target, identifier, value)
    e.target.classList.remove("pulse")
});

// Hover effects
saveIcon.addEventListener("mouseover", function(e) {
    e.target.classList.add("pulse")
});

saveIcon.addEventListener("mouseleave", function(e) {
    e.target.classList.remove("pulse")
});
```

f. **Fourth Column (Delete Icon):**
```javascript
trashIcon = document.createElement("i")
trashIcon.classList = "fa-solid fa-trash fa-lg save-icon"

// Similar event listeners as save icon
trashIcon.addEventListener("click", function(e) {
    identifier = e.target.parentElement.parentElement.children[0].children[0].value
    deleteBlock(e.target, identifier)
    e.target.classList.remove("pulse")
});
```

**5. Add New Entry Row:**

Creates a row with a "+" icon that, when clicked, adds an empty row for creating new blocks.

```javascript
let temp = document.getElementById("newRowTemplate");
let clon = temp.content.cloneNode(true);

clon.children[0].children[0].addEventListener("click", async function(e) {
    // Handler for adding new row
});

table.appendChild(clon);
```

**New Row Creation Logic:**

When "+" icon is clicked:
1. Checks if an empty row already exists (emptyRowActive flag)
2. If exists and unfilled, shows red border validation
3. If doesn't exist, creates new row with:
   - Editable input field for identifier
   - Editable textarea for value
   - Save icon (calls createNewValue)
   - Trash icon (calls deleteEmptyRow)
4. Inserts row before the "+" row

**6. Setup Auto-resize for All Textareas:**
```javascript
Array.from(document.getElementsByTagName("textarea")).forEach(element => {
    element.addEventListener("input", () => {
        element.style.height = baseHeight - 6 + "px";
        element.style.height = element.scrollHeight - 6 + "px"
    });
});
```

##### `saveValue(target, identifier, value)`
Updates an existing block's value on the server.

**Parameters:**
- `target` (HTMLElement): The save icon element (for visual feedback)
- `identifier` (string): The block identifier
- `value` (string): The new value to save

**Process:**
1. Changes icon to spinner:
```javascript
target.classList = "fa-solid fa-spinner fa-lg"
```

2. Sends PATCH request:
```javascript
sendRequest("PATCH", "update_block", {"extendedIdentifier": identifier, "value": value}, async function(error, response) {
    // Handle response
});
```

3. On success (error == null):
   - Changes icon color to green
   - Restores save icon
   - Waits 1 second
   - Resets icon color to black

4. On failure (error != null):
   - Changes icon color to red
   - Restores save icon
   - Waits 1 second
   - Resets icon color to black

**Visual Feedback Timeline:**
```
Click → Spinner → Success: Green (1s) → Black
              → Failure: Red (1s) → Black
```

##### `deleteBlock(target, identifier)`
Deletes a block from the server and removes its row from the UI.

**Parameters:**
- `target` (HTMLElement): The trash icon element
- `identifier` (string): The block identifier to delete

**Process:**
1. Changes icon to spinner
2. Sends POST request to delete_block endpoint
3. On success:
   - Removes the entire row from DOM:
   ```javascript
   target.parentElement.parentElement.remove()
   ```
4. On failure:
   - Shows red icon for 1 second
   - Restores trash icon

**Note:** Unlike saveValue, deleteBlock immediately removes the row on success without waiting.

##### `deleteEmptyRow(eventPointer)`
Removes the temporary empty row used for creating new entries.

**Parameters:**
- `eventPointer` (Event): The click event object

**Process:**
1. Removes the row from DOM:
```javascript
eventPointer.target.parentElement.parentElement.remove()
```

2. Resets flag:
```javascript
emptyRowActive = false
```

**Purpose:** Allows user to cancel new entry creation by clicking the trash icon.

##### `createNewValue(target, identifier, value, e)`
Creates a new block on the server and converts the temporary row to a permanent one.

**Parameters:**
- `target` (HTMLElement): The save icon element
- `identifier` (string): The new block identifier
- `value` (string): The block value
- `e` (function): The original event handler function (for removal)

**Validation:**
```javascript
if (identifier == "") {
    target.style["color"] = "red"
    target.parentElement.parentElement.children[0].style["border"] = "2px solid red"
    target.parentElement.parentElement.children[1].style["border"] = "2px solid black"
    target.parentElement.parentElement.children[0].children[0].focus();
    await sleep(1000)
    target.style["color"] = "black"
    return
}
```
- Validates that identifier is not empty
- Shows red border on identifier field if empty
- Focuses the identifier input
- Returns without creating block

**Creation Process:**

1. Shows spinner and disables identifier input:
```javascript
target.classList = "fa-solid fa-spinner fa-lg"
target.parentElement.parentElement.children[0].children[0].disabled = true
target.parentElement.parentElement.children[0].style["border"] = "2px solid black"
```

2. Sends POST request to create_block endpoint

3. On success:
   a. Changes icon to green
   b. Updates trash icon behavior:
      - Removes deleteEmptyRow listener
      - Adds deleteBlock listener with hover effects
   c. Updates save icon behavior:
      - Removes the original createNewValue listener
      - Adds saveValue listener
   d. Resets emptyRowActive flag
   e. Shows green for 1 second then resets to black

4. On failure:
   - Shows red icon for 1 second
   - Restores save icon (identifier remains editable)

**Row Conversion:**
The function effectively converts a temporary "new entry" row into a permanent data row by:
- Disabling the identifier input (making it read-only)
- Changing the trash icon from "delete row" to "delete block"
- Changing the save icon from "create" to "update"

---

## HTML Files

### login.html

**Purpose**: User authentication page with form for username and password input.

**Structure:**
```html
<!DOCTYPE html>
<html>
    <head>
        <script src="./login.js"></script>
        <link rel="stylesheet" href="login.css">
    </head>
    <body>
        <div class="backdrop"></div>
        <form class="login-box" id="login-form">
            <h1>Welcome to <i>EasySave</i></h1>
            <label for="username">Username</label>
            <input type="text" id="username" name="username" placeholder="johndoe" required>
            <label for="password">Password</label>
            <input type="password" id="password" name="password" placeholder="notyourbirthday123" required>
            <input type="submit" value="Login" id="submit">
            <a href="/register">Register</a>
        </form>
    </body>
</html>
```

**Key Elements:**

**backdrop:**
- Background image container
- CSS applies blur effect
- Fixed position behind main content

**login-box:**
- Main form container
- Flexbox layout for vertical alignment
- Target for mouse-tracking animation

**Form Inputs:**
- `username`: Text input with placeholder
- `password`: Password input (hidden characters)
- Both have `required` attribute for HTML5 validation

**Submit Button:**
- Styled as a button
- Triggers form submission event

**Register Link:**
- Navigation to registration page
- Styled to match overall design

---

### register.html

**Purpose**: New user registration page with form for username, email, and password.

**Structure:**
Similar to login.html with these differences:

**Additional Input:**
```html
<label for="email">Email</label>
<input type="text" id="email" name="email" placeholder="johndoe@gmail.com" required>
```

**Form ID:**
- `register-form` instead of `login-form`

**CSS Class:**
- `register-box` instead of `login-box`

**JavaScript:**
- Loads `register.js` instead of `login.js`

**Link:**
- Links to `/login` instead of `/register`

**Note:** Email input uses `type="text"` instead of `type="email"`, which means no built-in email validation. Consider changing to `type="email"` for better UX.

---

### main.html

**Purpose**: Main application interface for managing data blocks.

**Structure:**

**Head Section:**
```html
<head>
    <script src="./main.js"></script>
    <link rel="stylesheet" href="main.css">
    <script src="https://kit.fontawesome.com/da43a858c5.js" crossorigin="anonymous"></script>
</head>
```
- Loads main.js for functionality
- Loads main.css for styling
- Loads Font Awesome for icons

**Body Structure:**
```html
<body>
    <div class="backdrop"></div>
    <div class="main" id="main">
        <div class="header">
            <h1><strong id="username"></strong>'s Database</h1>
            <div>
                <i class="fa-solid fa-rotate-right fa-2xl" id="refreshIcon"></i>
                <i class="fa-solid fa-arrow-right-from-bracket fa-2xl" id="logoutIcon"></i>
            </div>
        </div>
        <table class="main-table">
        </table>
    </div>
</body>
```

**Key Elements:**

**Header:**
- Displays username (populated by JavaScript)
- Refresh icon (fa-rotate-right)
- Logout icon (fa-arrow-right-from-bracket)

**Table:**
- Initially empty
- Populated dynamically by refreshTable()
- Container for all data blocks

**Templates:**

**newRowTemplate:**
```html
<template id="newRowTemplate">
    <tr id="newEntryRow">
        <td><i class="fa-solid fa-plus"></i></td>
    </tr>
</template>
```
- Used to create the "add new entry" row
- Contains only a plus icon
- Cloned and inserted by JavaScript

**titlesTemplate:**
```html
<template id="titlesTemplate">
    <tr id="titles">
        <td>Identifier</td>
        <td>Value</td>
        <td></td>
        <td></td>
    </tr>
</template>
```
- Column headers for the table
- Last two columns are empty (for action icons)
- Cloned and inserted at table start

**HTML5 Template Element:**
- `<template>` tags are not rendered by browser
- Content is inert until cloned by JavaScript
- Efficient way to store HTML fragments

---

## CSS Files

### login.css

**Purpose**: Styles for the login page including layout, animations, and visual effects.

#### Layout Styles

**HTML Element:**
```css
html {
    background-color: blanchedalmond;
}
```
- Sets base background color

**Body:**
```css
body {
    display: flex;
    align-items: center;
    align-content: center;
    justify-content: center;
    width: 100vw;
    height: 100vh;
    margin: 0;
    color: white;
    font-family: 'Trebuchet MS', ...;
}
```
- Flexbox centering for login box
- Full viewport dimensions
- White text color
- Sans-serif font stack

#### Background

**backdrop:**
```css
.backdrop {
    background-image: url("https://static.vecteezy.com/system/resources/thumbnails/059/607/113/small_2x/yellow-spheres-composition-on-black-backdrop-cut-out-transparent-png.png");
    background-repeat: no-repeat;
    background-position: center center;
    background-size: cover;
    filter: blur(5px);
    position: fixed;
    width: 100vw;
    height: 100vh;
    z-index: -1;
}
```
- External image URL
- Centered and covering full viewport
- 5px blur for depth effect
- Fixed position behind content (z-index: -1)

#### Login Box

**login-box:**
```css
.login-box {
    width: 36vw;
    height: auto;
    padding: 2vw 2vw 2vh 2vw;
    background-color: coral;
    border-radius: calc(1vw + 1vh);
    box-shadow: 0 0 0 0 rgba(0, 0, 0, 1);
    transform: translate(var(--tx, 0), var(--ty, 0)) scale(1);
    will-change: transform;
    animation: pulse 2s infinite;
    display: flex;
    justify-content: left;
    flex-direction: column;
}
```

**Key Properties:**
- **Width**: 36% of viewport width (responsive)
- **Padding**: Viewport-based for responsiveness
- **Background**: Coral color (consistent with brand)
- **Border Radius**: Calculated as sum of 1vw + 1vh (more rounded on larger screens)
- **Transform**: Uses CSS variables --tx and --ty (set by JavaScript)
- **will-change**: Optimizes for transform animations
- **Animation**: Continuous pulse effect

**Hover Effect:**
```css
.login-box:hover {
    transform: rotateY(180deg);
}
```
- Flips box on Y-axis when hovered
- **Note**: This may interfere with the mouse-tracking animation and cause UI issues. Consider removing or modifying.

#### Animations

**Pulse Animation:**
```css
@keyframes pulse {
    0% {
        transform: translate(var(--tx, 0), var(--ty, 0)) scale(1);
        box-shadow: 0 0 0 0 rgba(255, 201, 121, 1);
    }
    70% {
        transform: translate(var(--tx, 0), var(--ty, 0)) scale(1);
        box-shadow: 0 0 0 30px rgba(255, 201, 121, 0);
    }
    100% {
        transform: translate(var(--tx, 0), var(--ty, 0)) scale(1);
        box-shadow: 0 0 0 0 rgba(255, 201, 121, 0);
    }
}
```

**Effect:**
- Creates expanding ring effect
- Ring starts visible (opacity 1) and expands to 30px while fading to transparent
- Maintains the translate transformation from JavaScript
- 2-second cycle, infinite repeat

**Visual Flow:**
- 0-70%: Ring expands from 0px to 30px, fading out
- 70-100%: Ring at 30px, fully transparent
- Reset: Jump back to 0%, fully visible

#### Form Elements

**Labels:**
```css
label {
    margin-top: 0;
    margin-left: 1vw;
    margin-bottom: 1vh;
}
```
- Slight left margin for alignment
- Vertical spacing with margin-bottom

**Inputs:**
```css
input {
    margin-bottom: 3vh;
    height: 3vh;
    border-radius: calc(0.5vw + 0.5vh);
    padding-left: 2vw;
    border: 0;
    font-size: 1em;
}
```
- 3vh bottom margin (spacing between fields)
- Rounded corners (responsive)
- Left padding for text
- No border for clean look

**Submit Button:**
```css
#submit {
    width: 40%;
    margin-left: 30%;
    margin-right: 30%;
    padding: 1%;
    margin-top: 1vh;
    margin-bottom: 0;
    background-color: royalblue;
    color: white;
    font-weight: bold;
    height: 4vh;
    cursor: pointer;
}
```
- Centered with margins (30% left, 30% right)
- Royal blue background (distinct from coral)
- White bold text
- Pointer cursor on hover
- Slightly taller than text inputs (4vh vs 3vh)

---

### register.css

**Purpose**: Styles for the registration page.

**Differences from login.css:**
- `.register-box` class instead of `.login-box`
- Identical styling otherwise

**Code Duplication:**
All CSS rules are duplicated between login.css and register.css except for the class name. Consider consolidating common styles in a shared CSS file and keeping only page-specific styles separate.

**Potential Refactoring:**
```css
/* shared.css */
.auth-box { /* common styles */ }

/* login.css */
.login-box { @extend .auth-box; }

/* register.css */
.register-box { @extend .auth-box; }
```

---

### main.css

**Purpose**: Styles for the main database management interface.

#### Layout

**Body:**
```css
body {
    display: flex;
    align-items: center;
    align-content: center;
    justify-content: center;
    background-color: blanchedalmond;
    width: 100dvw;   /* dvw instead of vw */
    height: 100dvh;  /* dvh instead of vh */
    margin: 0;
    color: white;
    font-family: 'Trebuchet MS', ...;
}
```
- Uses dynamic viewport units (dvw, dvh)
- Better handling for mobile browsers with dynamic toolbars

**Main Container:**
```css
.main {
    width: 93dvw;
    height: 93dvh;
    background-color: coral;
    padding: 1vh;
    border-radius: calc(1vw + 1vh);
    display: flex;
    justify-content: left;
    flex-direction: column;
    overflow: hidden;
}
```
- Nearly full viewport (93% to leave small margin)
- Flexbox column layout
- Overflow hidden (scrolling handled by table)

#### Header

**header:**
```css
.header {
    height: 10%;
    padding-left: 2vw;
    padding-right: 2vw;
    width: calc(100% - 2*2vw);
    background-color: rgb(255, 200, 180);
    border-top-left-radius: calc(1vw);
    border-top-right-radius: calc(1vw);
    display: flex;
    color: black;
    align-items: center;
    justify-content: space-between;
}
```
- 10% of main container height
- Lighter coral background (distinguishes from main area)
- Rounded top corners
- Flexbox with space-between (username left, icons right)
- Black text color (contrast with light background)

**header h1:**
```css
.header h1 {
    font-weight: normal;
}
```
- Normal weight (only username is bold via `<strong>` tag)

#### Icons

**General Icon:**
```css
i {
    cursor: pointer;
}
```
- All icons are clickable

**Save and Delete Icons:**
```css
.fa-floppy-disk {
    transform: scale(1.2);
}

.fa-trash {
    transform: scale(1.2);
}
```
- Scaled up 20% for better visibility

#### Animations

**Spin Animation:**
```css
.spin {
    animation: rotate 0.5s;
}

@keyframes rotate {
    0% { transform: rotate(0); }
    100% { transform: rotate(360deg); }
}
```
- Single 360° rotation in 0.5 seconds
- Applied to refresh icon when clicked

**Pulse Animation:**
```css
.pulse {
    animation: pulse 1.5s infinite ease-in-out;
}

@keyframes pulse {
    0% { transform: scale(1.2); }
    50% { transform: scale(1.3); }
    100% { transform: scale(1.2); }
}
```
- Oscillates between 1.2x and 1.3x scale
- 1.5-second cycle
- Infinite repeat
- Applied on hover for save/delete icons

**Spinner Animation:**
```css
.fa-spinner {
    animation: rotate 2s infinite linear;
}
```
- Continuous rotation
- 2-second per rotation
- Infinite repeat
- Linear timing (constant speed)

#### Table Styles

**main-table:**
```css
.main-table {
    width: calc(100% - 5vw);
    margin-top: 3vh;
    padding-left: 0.5vw;
    margin-left: 2vw;
    padding-right: 0.5vw;
    margin-right: 2vw;
    color: black;
    font-size: 1.5em;
    background-color: bisque;
    border-radius: 10px;
}
```
- Takes full width minus margins
- Bisque background (light tan color)
- Large font size for readability
- Fixed border radius (not responsive)

**Table Rows:**
```css
.main-table tr {
    display: flex;
    justify-content: center;
    align-items: flex-start;
    margin-top: 1vh;
}

.main-table tr:last-child {
    margin-bottom: 1vh;
}

.main-table tr:nth-child(2) {
    margin-top: 0;
}
```
- Flexbox layout for row cells
- Vertical spacing between rows
- First data row (second child) has no top margin
- Last row has bottom margin

**Table Cells:**
```css
.main-table td {
    display: flex;
    align-items: center;
    width: 100%;
    height: auto;
    min-height: 33px;
    overflow: hidden;
    padding: 0;
    border: 2px solid black;
    border-radius: 10px;
}
```
- Flexbox for centering content
- Auto height with minimum
- Hidden overflow
- Black border with rounded corners

**Column Sizing:**
```css
.main-table td:first-child {
    width: 40%;
    margin-right: 1vw;
}

.main-table td:nth-last-child(-n + 2) {
    width: 7%;
    border: none;
    margin-left: 0.6vw;
    height: 40px;
    overflow: visible;
}
```
- First column (identifier): 40% width
- Last two columns (icons): 7% width each, no border
- Remaining width automatically distributed to value column

**Title Row:**
```css
tr#titles td {
    border: none;
    margin-left: 2vw;
}
```
- No borders for header row
- Left margin for alignment

#### Form Elements in Table

**Textareas:**
```css
.main-table textarea {
    width: 100%;
    height: 27px;
    border: none;
    padding-left: 2vw;
    font-size: 1.1em;
    white-space: pre-wrap;
    word-wrap: break-word;
    padding-top: 3px;
    padding-bottom: 3px;
    outline: none;
    resize: none;
    padding-right: 1vw;
}
```
- Full width of cell
- Initial height 27px (grows with content via JavaScript)
- Pre-wrap: preserves whitespace and line breaks
- Word-wrap: breaks long words
- No resize handle (JavaScript handles auto-resize)
- No outline (focus ring removed)

**Inputs:**
```css
.main-table input {
    height: 27px;
    border: none;
    padding-left: 2vw;
    font-size: 1.1em;
    white-space: pre-wrap;
    word-wrap: break-word;
    padding-top: 3px;
    padding-bottom: 3px;
    outline: none;
}
```
- Similar to textarea
- Fixed height (doesn't auto-grow)
- Used for identifier field

**General Border Styling:**
```css
th, td {
    border: 2px solid black;
    border-radius: 10px;
}
```
- Applied to all cells
- Overridden for specific cases (icons, titles)

---

## Additional Notes

### Performance Considerations

**Animation Performance:**
- Mouse tracking uses `requestAnimationFrame` for 60fps
- `will-change: transform` property hints browser for optimization
- Passive event listeners for scroll performance

**DOM Manipulation:**
- Uses `innerHTML = ""` to clear table (fast but loses event listeners)
- Creates elements with `createElement` (more verbose but better performance)
- Uses templates for repeating structures

### Security Considerations

**Cookies:**
- Not using `HttpOnly` flag (JavaScript can access)
- Not using `Secure` flag (can be sent over HTTP)
- Using `SameSite=Lax` (some CSRF protection)

**API Communication:**
- Credentials sent in headers (better than query params)
- No CSRF token implementation
- No rate limiting on client side

**Input Validation:**
- HTML5 `required` attribute (client-side only)
- No sanitization before sending to API
- No XSS protection (relies on API/backend)

### Accessibility Considerations

**Issues:**
- Icons lack aria-labels
- No keyboard navigation support
- No focus indicators (outline: none)
- Color-only feedback (red/green)
- No screen reader support

**Improvements Needed:**
- Add ARIA labels to icons
- Implement keyboard shortcuts
- Add focus styles
- Add text alternatives for color feedback
- Add role attributes to table

### Browser Compatibility

**Modern Features Used:**
- CSS variables (--tx, --ty)
- Template elements
- Arrow functions
- Async/await
- Flexbox
- Dynamic viewport units (dvw, dvh)

**Minimum Browser Versions:**
- Chrome 90+
- Firefox 88+
- Safari 14+
- Edge 90+

### Mobile Responsiveness

**Current State:**
- Viewport units provide some responsiveness
- Fixed percentage widths (36vw, 40%, etc.)
- No media queries for different screen sizes
- Touch events not explicitly handled

**Limitations:**
- Login box may be too small on mobile
- Table columns may overflow on narrow screens
- Icons may be too small for touch targets
- No mobile-specific layouts

### Code Quality

**Strengths:**
- Consistent naming conventions
- Clear separation of concerns
- Reusable functions
- Visual feedback for all actions

**Areas for Improvement:**
- Code duplication (getCookie, animation code)
- Global variables in main.js
- Long functions (refreshTable)
- Limited error handling
- No input sanitization
- Hardcoded API URL

### Maintenance Recommendations

1. **Extract shared code** into common.js (getCookie, animation code)
2. **Create configuration file** for API URL and settings
3. **Add comprehensive error handling** with user-friendly messages
4. **Implement loading states** for all async operations
5. **Add unit tests** for utility functions
6. **Add integration tests** for API interactions
7. **Implement proper logging** for debugging
8. **Create style guide** for consistent CSS
9. **Consolidate CSS** to reduce duplication
10. **Add JSDoc comments** for better IDE support

---

## Glossary

**DOM**: Document Object Model - tree representation of HTML
**XHR**: XMLHttpRequest - API for making HTTP requests
**Lerp**: Linear interpolation - smooth transition between two values
**Viewport**: The visible area of the web page
**CSRF**: Cross-Site Request Forgery - security vulnerability
**XSS**: Cross-Site Scripting - security vulnerability
**ARIA**: Accessible Rich Internet Applications - accessibility standard
**CDN**: Content Delivery Network - distributed server network for resources

---

## Revision History

- **Version 1.0** (Current): Initial detailed documentation
- Created comprehensive code documentation for all files
- Documented API endpoints and data flow
- Identified security and accessibility issues
- Provided maintenance recommendations
