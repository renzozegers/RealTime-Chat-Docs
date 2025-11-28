# 📖 Glossary of Terms

A complete dictionary of all technical terms used in this chat service. Perfect for beginners who encounter unfamiliar words!

## 🔤 Terms A-Z

### A

**API (Application Programming Interface)**
- **Simple:** A way for programs to talk to each other
- **Analogy:** Like a waiter taking your order and bringing food
- **In this project:** The endpoints that let your frontend talk to the chat server

**Async/Await**
- **Simple:** A way to wait for something to finish before continuing
- **Analogy:** Like waiting for a pizza to bake before eating it
- **In this project:** Used when waiting for database queries to finish

**Authentication**
- **Simple:** Proving who you are (like showing ID)
- **Analogy:** Like showing your ID card to enter a building
- **In this project:** Using JWT tokens to prove you're logged in

**Authorization**
- **Simple:** Checking if you're allowed to do something
- **Analogy:** Like checking if you have permission to enter a VIP area
- **In this project:** Checking if you can delete a message or add users to groups

---

### B

**Backend**
- **Simple:** The part of the app that runs on a server (not your computer)
- **Analogy:** Like the kitchen in a restaurant (you don't see it, but it does the work)
- **In this project:** The chat server code that processes messages

**Blocking**
- **Simple:** Preventing someone from contacting you
- **Analogy:** Like blocking someone's phone number
- **In this project:** Users can block each other, preventing messages

**Broadcast**
- **Simple:** Sending the same message to multiple people at once
- **Analogy:** Like a TV station broadcasting to all viewers
- **In this project:** Sending a group message to all members

---

### C

**Cache**
- **Simple:** Temporary storage for frequently used data (faster access)
- **Analogy:** Like keeping your phone number on speed dial instead of looking it up each time
- **In this project:** Storing recent messages and user info in memory for fast access

**Client**
- **Simple:** Your browser or app (the thing that uses the service)
- **Analogy:** Like a customer in a restaurant
- **In this project:** The frontend app that users interact with

**Connection**
- **Simple:** The link between your browser and the server
- **Analogy:** Like a phone call staying open
- **In this project:** WebSocket connection that stays open for real-time chat

**CORS (Cross-Origin Resource Sharing)**
- **Simple:** Browser security that controls which websites can connect to the server
- **Analogy:** Like a bouncer checking if you're on the guest list
- **In this project:** Security headers that allow your frontend to connect

---

### D

**Database**
- **Simple:** Permanent storage for data (like a filing cabinet)
- **Analogy:** Like a library that stores all books permanently
- **In this project:** PostgreSQL database stores all messages, users, groups

**Data Structure**
- **Simple:** A way to organize data (like a list, table, or map)
- **Analogy:** Like different ways to organize files (folders, labels, etc.)
- **In this project:** Maps, Sets, Arrays used to organize connections and messages

**Decryption**
- **Simple:** Converting encrypted data back to readable text
- **Analogy:** Like unlocking a safe to get your valuables
- **In this project:** Converting encrypted messages back to readable text

**DM (Direct Message)**
- **Simple:** Private message between two people
- **Analogy:** Like a text message between you and one friend
- **In this project:** Private messages sent between two users

---

### E

**Encryption**
- **Simple:** Converting readable text into secret code for security
- **Analogy:** Like putting a letter in a safe before mailing it
- **In this project:** Messages are encrypted before storing in database

**Endpoint**
- **Simple:** A specific operation the server can perform
- **Analogy:** Like a menu item in a restaurant
- **In this project:** Like "send_private_message" or "get_online_users"

**Error Handling**
- **Simple:** What the code does when something goes wrong
- **Analogy:** Like having a backup plan when Plan A fails
- **In this project:** Try/catch blocks that handle errors gracefully

**Event**
- **Simple:** Something that happens (like receiving a message)
- **Analogy:** Like a doorbell ringing (the event is the ring)
- **In this project:** Message received, user connected, user disconnected

**Event Queue**
- **Simple:** A waiting list for events when user is offline
- **Analogy:** Like a mailbox holding mail until you check it
- **In this project:** Messages stored for offline users, delivered when they reconnect

---

### F

**Frontend**
- **Simple:** The part of the app you see and interact with
- **Analogy:** Like the dining room in a restaurant (where customers sit)
- **In this project:** The browser app that shows the chat interface

**Function**
- **Simple:** A reusable piece of code that does something
- **Analogy:** Like a recipe that you can use multiple times
- **In this project:** Functions like `sendMessage()`, `handleAuth()`, etc.

---

### G

**Git**
- **Simple:** A tool that tracks changes to code
- **Analogy:** Like a time machine for your files
- **In this project:** Used to manage code versions and push to GitHub

**Group Chat**
- **Simple:** Messages sent to multiple people at once
- **Analogy:** Like a group text message
- **In this project:** Chat rooms with multiple members

---

### H

**Handler**
- **Simple:** Code that processes a specific type of request
- **Analogy:** Like a specialist doctor for specific problems
- **In this project:** `handleSendMessage()`, `handleAuthentication()`, etc.

**HTTP (HyperText Transfer Protocol)**
- **Simple:** The language websites use to communicate
- **Analogy:** Like the postal service for the web
- **In this project:** Used for HTTP endpoints (health check, profile pictures)

**HTTPS**
- **Simple:** Secure HTTP (encrypted)
- **Analogy:** Like sending a letter in a locked box instead of a regular envelope
- **In this project:** Secure version of HTTP with TLS encryption

---

### I

**IP Address (Internet Protocol Address)**
- **Simple:** Unique identifier for each device on the internet
- **Analogy:** Like your home address for the internet
- **In this project:** Used to identify where connections come from and rate limit

**Index**
- **Simple:** A way to find data quickly in a database
- **Analogy:** Like an index in a book (helps you find pages quickly)
- **In this project:** Database indexes speed up user lookups

**Input Validation**
- **Simple:** Checking if user input is correct and safe
- **Analogy:** Like checking if someone's ID is valid before letting them in
- **In this project:** Validating message content, user IDs, etc.

---

### J

**JavaScript**
- **Simple:** A programming language for websites and servers
- **Analogy:** Like the language you use to tell a computer what to do
- **In this project:** The programming language this chat server is written in

**JSON (JavaScript Object Notation)**
- **Simple:** A way to format data (like a list or form)
- **Analogy:** Like filling out a standardized form
- **In this project:** All messages are sent as JSON format

**JWT (JSON Web Token)**
- **Simple:** A secure way to prove who you are without storing passwords
- **Analogy:** Like a special ticket that proves you're allowed in (without revealing your password)
- **In this project:** Used for authentication (proves you're logged in)

**JTI (JWT ID)**
- **Simple:** Unique ID for each JWT token
- **Analogy:** Like a serial number on a ticket
- **In this project:** Used to revoke tokens (logout)

---

### K

**Keep-Alive**
- **Simple:** Sending small messages to keep connection alive
- **Analogy:** Like saying "I'm still here!" during a phone call
- **In this project:** Ping/pong messages keep WebSocket connections alive

---

### L

**Latency**
- **Simple:** Time it takes for data to travel (delay)
- **Analogy:** Like how long it takes a letter to arrive
- **In this project:** Time between sending and receiving a message

**Load Balancing**
- **Simple:** Distributing work across multiple servers
- **Analogy:** Like having multiple cashiers at a store to serve more customers
- **In this project:** Not used yet, but can be added to handle more users

**Logging**
- **Simple:** Writing down what happens (for debugging)
- **Analogy:** Like keeping a diary of events
- **In this project:** Console.log() statements that help developers see what's happening

---

### M

**Map (Data Structure)**
- **Simple:** A way to store data with keys (like a dictionary)
- **Analogy:** Like a phone book (name = key, number = value)
- **In this project:** Used to store clients, conversations, groups

**Message**
- **Simple:** A piece of data sent from one place to another
- **Analogy:** Like a letter you send through mail
- **In this project:** Chat messages between users

**Message Router**
- **Simple:** Code that decides which handler to use for each message
- **Analogy:** Like a receptionist directing you to the right department
- **In this project:** Routes messages by type to appropriate handlers

**Microservice**
- **Simple:** A small, independent service that does one thing
- **Analogy:** Like a specialist shop (just sells coffee) vs. a department store
- **In this project:** Chat service is separate from the main app (Java backend)

**Middleware**
- **Simple:** Code that runs before the main handler
- **Analogy:** Like security check before entering a building
- **In this project:** Authentication, rate limiting, validation

---

### N

**Node.js**
- **Simple:** A way to run JavaScript on servers (not just browsers)
- **Analogy:** Like JavaScript getting a new superpower (can work on servers too)
- **In this project:** The runtime that runs this chat server

**Normalize**
- **Simple:** Converting data to a standard format
- **Analogy:** Like converting all phone numbers to the same format
- **In this project:** Converting user IDs to strings consistently

---

### O

**Object**
- **Simple:** A way to group related data together
- **Analogy:** Like a form with multiple fields (name, age, email)
- **In this project:** Message objects, user objects, etc.

**Offline**
- **Simple:** Not connected to the server
- **Analogy:** Like being out of phone signal range
- **In this project:** Users who aren't connected get messages queued for later

**Online**
- **Simple:** Connected to the server right now
- **Analogy:** Like being on an active phone call
- **In this project:** Users currently connected via WebSocket

**Origin**
- **Simple:** Where a request comes from (which website)
- **Analogy:** Like return address on a letter
- **In this project:** Used for CORS (security check)

---

### P

**Parameterized Query**
- **Simple:** A safe way to run database queries (prevents hacking)
- **Analogy:** Like using a locked box instead of an open envelope
- **In this project:** All database queries use parameters to prevent SQL injection

**Payload**
- **Simple:** The actual data being sent
- **Analogy:** Like the contents of a package (not the box itself)
- **In this project:** The message content, user data, etc. inside requests

**Ping/Pong**
- **Simple:** Small messages to check if connection is alive
- **Analogy:** Like saying "Are you there?" and getting "Yes!" back
- **In this project:** Keeps WebSocket connections alive and detects dead connections

**Pool (Connection Pool)**
- **Simple:** A group of reusable database connections
- **Analogy:** Like a carpool (reusing vehicles instead of buying new ones)
- **In this project:** Database connection pool reuses connections for efficiency

**Port**
- **Simple:** A door number on a server
- **Analogy:** Like apartment number (3001) at an address
- **In this project:** Server runs on port 3001

**PostgreSQL**
- **Simple:** A type of database (permanent data storage)
- **Analogy:** Like a high-quality filing cabinet for data
- **In this project:** Database that stores all messages, users, groups

**Protocol**
- **Simple:** Rules for how computers communicate
- **Analogy:** Like the rules of a language (grammar, vocabulary)
- **In this project:** WebSocket protocol for real-time communication

---

### Q

**Query**
- **Simple:** A question asked to the database
- **Analogy:** Like asking "Who lives here?" at an address
- **In this project:** Database queries like "Get user with ID 123"

**Queue**
- **Simple:** A waiting line (first in, first out)
- **Analogy:** Like a line at a store checkout
- **In this project:** Event queue holds messages for offline users

---

### R

**Rate Limiting**
- **Simple:** Limiting how many requests someone can make
- **Analogy:** Like limiting how many times you can ring a doorbell per minute
- **In this project:** Prevents spam (max 240 messages per minute per user)

**Redis**
- **Simple:** A fast in-memory database for caching
- **Analogy:** Like a super-fast temporary notepad (much faster than regular storage)
- **In this project:** Optional cache for faster message lookups

**Request**
- **Simple:** Asking the server to do something
- **Analogy:** Like ordering food at a restaurant
- **In this project:** Client requests like "send message" or "get online users"

**Response**
- **Simple:** The server's answer to a request
- **Analogy:** Like receiving your food order
- **In this project:** Server responses like "message sent" or list of online users

**Router**
- **Simple:** Code that decides where messages go
- **Analogy:** Like a mail sorter directing letters to correct departments
- **In this project:** messageRouter.js routes messages by type to handlers

---

### S

**Server**
- **Simple:** A computer that provides a service
- **Analogy:** Like a restaurant that serves food to customers
- **In this project:** The Node.js server that handles chat messages

**Session**
- **Simple:** A period of being connected/active
- **Analogy:** Like your visit to a store (start when you enter, end when you leave)
- **In this project:** WebSocket session from connection open to close

**Set (Data Structure)**
- **Simple:** A collection of unique items (no duplicates)
- **Analogy:** Like a guest list (each name appears only once)
- **In this project:** Used for participants, online users (no duplicates)

**SQL (Structured Query Language)**
- **Simple:** Language for talking to databases
- **Analogy:** Like the language you use to ask a librarian for books
- **In this project:** Used to store and retrieve messages from PostgreSQL

**SQL Injection**
- **Simple:** A type of hack where malicious code is inserted into database queries
- **Analogy:** Like someone slipping a fake order into your restaurant order system
- **In this project:** Prevented by using parameterized queries

**State**
- **Simple:** Current condition or data
- **Analogy:** Like your current location (state = at home, at work, etc.)
- **In this project:** Server maintains state of connections, online users, conversations

**Stateless**
- **Simple:** Not remembering previous requests
- **Analogy:** Like a vending machine (doesn't remember what you bought before)
- **In this project:** HTTP endpoints are stateless (each request is independent)

**Stateful**
- **Simple:** Remembering previous requests (maintaining connection)
- **Analogy:** Like a conversation with a friend (remembers what you talked about)
- **In this project:** WebSocket connections are stateful (server remembers you)

---

### T

**TCP/IP**
- **Simple:** The foundation of internet communication
- **Analogy:** Like the postal service system for the internet
- **In this project:** WebSocket uses TCP/IP for connections

**Terminal/Command Line**
- **Simple:** A text-based way to control your computer
- **Analogy:** Like giving instructions to a computer by typing instead of clicking
- **In this project:** Used to run the server (like `node server.js`)

**Timeout**
- **Simple:** Automatically closing connection after inactivity
- **Analogy:** Like a phone call ending if nobody talks for 5 minutes
- **In this project:** Connections timeout after 5 minutes of no activity

**Token**
- **Simple:** A secure proof of identity (like a ticket)
- **Analogy:** Like a wristband at an event (proves you paid)
- **In this project:** JWT tokens prove you're authenticated

**Transaction**
- **Simple:** Multiple operations that succeed or fail together
- **Analogy:** Like buying groceries (all items succeed or all fail together)
- **In this project:** Database transactions ensure data consistency

---

### U

**URL (Uniform Resource Locator)**
- **Simple:** An address for a resource on the internet
- **Analogy:** Like a street address for a website
- **In this project:** `ws://localhost:3001` is the WebSocket URL

**UUID (Universally Unique Identifier)**
- **Simple:** A unique ID that's almost impossible to duplicate
- **Analogy:** Like a fingerprint (everyone has a unique one)
- **In this project:** Used for client IDs, message IDs

---

### V

**Validation**
- **Simple:** Checking if data is correct and safe
- **Analogy:** Like checking if someone's ID is valid before letting them in
- **In this project:** Validating message content, user IDs, etc.

**Variable**
- **Simple:** A named container that stores a value
- **Analogy:** Like a labeled box that holds something
- **In this project:** Variables like `message`, `userId`, `client`

---

### W

**WebSocket**
- **Simple:** A persistent connection that allows instant two-way communication
- **Analogy:** Like a phone call (stays open, both sides can talk anytime)
- **In this project:** Used for real-time chat (instant message delivery)

**WebSocket Server**
- **Simple:** Server that handles WebSocket connections
- **Analogy:** Like a phone operator that connects calls
- **In this project:** The wss (WebSocket Server) that handles all connections

---

### X, Y, Z

**XSS (Cross-Site Scripting)**
- **Simple:** A security attack where malicious code is injected into websites
- **Analogy:** Like someone slipping a fake instruction into your recipe book
- **In this project:** Prevented by input validation and sanitization

---

## 🔄 Related Terms

### Authentication vs Authorization
- **Authentication:** "Who are you?" (proving identity)
- **Authorization:** "Can you do this?" (checking permissions)

### HTTP vs WebSocket
- **HTTP:** Request-response, connection closes after each request
- **WebSocket:** Persistent connection, bidirectional, real-time

### Client vs Server
- **Client:** Your browser/app (requests things)
- **Server:** Remote computer (provides things)

### Frontend vs Backend
- **Frontend:** What users see and interact with (browser)
- **Backend:** What runs on server (processes requests)

### Database vs Cache
- **Database:** Permanent storage (like a filing cabinet)
- **Cache:** Temporary storage for speed (like a desk drawer)

---

## 📚 Common Phrases Explained

**"In this project"** = How this term is used in the chat service code

**"Analogy"** = A real-world comparison to help understand

**"Simple"** = Beginner-friendly explanation

---

## 💡 How to Use This Glossary

1. **While reading guides:** Keep this glossary open in another tab
2. **Encounter unfamiliar word?** Search for it here (Ctrl+F or Cmd+F)
3. **Still confused?** Read the analogy - it helps!
4. **Need more detail?** Check the guide that mentions it

---

## 🎯 Quick Reference by Topic

### Networking Terms
- IP Address, Port, TCP/IP, URL, Origin, Latency, Protocol

### Programming Terms
- JavaScript, Function, Variable, Object, Array, Map, Set, JSON

### Database Terms
- Database, PostgreSQL, Query, SQL, Index, Transaction, Pool

### Security Terms
- JWT, Authentication, Authorization, Encryption, Decryption, CORS, SQL Injection

### Chat-Specific Terms
- WebSocket, Message, DM, Group Chat, Broadcast, Offline, Online, Ping/Pong

---

**Still confused?** Check the [What Things Are](./WHAT_THINGS_ARE.md) guide for more detailed explanations!

**Want more detail?** Read the specific guide that uses that term - each guide explains terms in context.

