# Social Network for Parents - API and Data Model Design

## Problem Context

We are building a social network for parents where they can be connected to other parents based on two primary factors:

1. **School their child goes to** (Mandatory - Parents must upload their child’s school ID card to sign up).
2. **Society/Community they live in** (Optional - Parents can choose to skip uploading their address during signup).

For example, if a child goes to `DPS School in Grade I, Section F` and the parent lives in `Brigade Society`, they will automatically be joined to 5 "social circles":
- DPS School
- Class I, DPS School
- Section F, Class I, DPS School
- Brigade Society
- Brigade Society, DPS School

If a parent has not confirmed where they live, only the first three social circles (related to the school) will be created. These social circles are "auto-joined" for all new parents. The social graph has two root nodes: **school** and **society/community**, from which all social circles descend.

---

## Problem Statement

### Problem 1: Data Model, API & Efficient Retrieval

When onboarding a parent, the relevant social circles must be created if they don’t exist, and the parent should be added to these circles. We need to design:
1. A database schema to store and retrieve these circles efficiently.
2. APIs to handle the creation of circles and adding parents to these circles.

### Problem 2: Parent Posting to a Circle

A parent belonging to a social circle should be able to post and reply within that circle. This includes:
1. Posting to a circle.
2. Posting replies to existing posts.
3. Replying to replies (threading like Slack, not infinite nesting).
4. Upvoting or downvoting posts and replies.

### Problem 3: Schema Evolution

As time passes, children may change grades, and parents might initiate new social circles. We need a plan for handling:
1. Children changing grades.
2. Parent-initiated circles.

### Problem 4: Enhancements for New Circles

Parents should be able to create new circles based on existing ones (e.g., "Brigade Society Bus No. 92" under "Brigade Society"). We need a mechanism to allow discoverability of these new circles.

---

## Solution Design

### Problem 1: Data Model, API & Efficient Retrieval

#### Data Model

We'll use a relational database schema with the following tables:

1. **`schools`**
   - `id` (Primary Key)
   - `name` (e.g., "DPS School")

2. **`classes`**
   - `id` (Primary Key)
   - `school_id` (Foreign Key to `schools`)
   - `name` (e.g., "Class I")

3. **`sections`**
   - `id` (Primary Key)
   - `class_id` (Foreign Key to `classes`)
   - `name` (e.g., "Section F")

4. **`societies`**
   - `id` (Primary Key)
   - `name` (e.g., "Brigade Society")

5. **`social_circles`**
   - `id` (Primary Key)
   - `type` (Enum: School, Class, Section, Society)
   - `school_id` (Foreign Key to `schools`, Nullable)
   - `class_id` (Foreign Key to `classes`, Nullable)
   - `section_id` (Foreign Key to `sections`, Nullable)
   - `society_id` (Foreign Key to `societies`, Nullable)

6. **`parents`**
   - `id` (Primary Key)
   - `name`
   - `child_school_id` (Foreign Key to `schools`)
   - `child_class_id` (Foreign Key to `classes`)
   - `child_section_id` (Foreign Key to `sections`)
   - `society_id` (Foreign Key to `societies`, Nullable)

7. **`circle_memberships`**
   - `id` (Primary Key)
   - `parent_id` (Foreign Key to `parents`)
   - `social_circle_id` (Foreign Key to `social_circles`)

#### API Design

1. **Endpoint:** `POST /api/circles/onboard`
   - **Request:**
     ```json
     {
       "parent_id": 123,
       "child_school_id": 1,
       "child_class_id": 1,
       "child_section_id": 1,
       "society_id": 1
     }
     ```
   - **Process:**
     - Check if the relevant social circles exist.
     - If not, create them (e.g., school circle, class circle, section circle).
     - Add the parent to the appropriate circles by inserting rows into the `circle_memberships` table.

2. **Implementation Logic:**
   - First, query the `social_circles` table using the provided `school_id`, `class_id`, and `section_id` to check if circles exist.
   - If a circle doesn’t exist, insert a new circle.
   - Create a membership in all required circles (`school`, `class`, `section`, and optionally `society`).

---

### Problem 2: Parent Posting to a Circle

#### Data Model

1. **`posts`**
   - `id` (Primary Key)
   - `parent_id` (Foreign Key to `parents`)
   - `social_circle_id` (Foreign Key to `social_circles`)
   - `content`
   - `created_at`

2. **`replies`**
   - `id` (Primary Key)
   - `post_id` (Foreign Key to `posts`)
   - `parent_id` (Foreign Key to `parents`)
   - `reply_to` (Nullable, to represent thread replies)
   - `content`
   - `created_at`

3. **`votes`**
   - `id` (Primary Key)
   - `post_id` (Nullable, Foreign Key to `posts`)
   - `reply_id` (Nullable, Foreign Key to `replies`)
   - `parent_id` (Foreign Key to `parents`)
   - `vote_type` (Enum: Upvote, Downvote)

#### API Design

1. **Endpoint:** `POST /api/circles/:circle_id/posts`
   - **Request:**
     ```json
     {
       "parent_id": 123,
       "content": "Kids getting sick after eating lunch"
     }
     ```

2. **Endpoint:** `POST /api/posts/:post_id/replies`
   - **Request:**
     ```json
     {
       "parent_id": 123,
       "content": "I agree, my kid also had stomach issues",
       "reply_to": null
     }
     ```

3. **Endpoint:** `POST /api/replies/:reply_id/replies`
   - **Request:**
     ```json
     {
       "parent_id": 123,
       "content": "I heard it could be due to the new supplier",
       "reply_to": 234
     }
     ```

4. **Endpoint:** `POST /api/posts/:post_id/votes`
   - **Request:**
     ```json
     {
       "parent_id": 123,
       "vote_type": "Upvote"
     }
     ```

---

### Problem 3: Schema Evolution

1. **Children Changing Grades:**
   - When a child moves from one grade to another, we need to update the parent's `child_class_id` and `child_section_id` in the `parents` table.
   - Automatically reassign the parent to the new class and section circles by adjusting the `circle_memberships` entries.

2. **Parent-Initiated Circles:**
   - Allow parents to create circles based on interests or additional criteria.
   - Add an `is_custom` column to `social_circles` and a `created_by` field to track which parent created it.

---

### Problem 4: Enhancements for New Circles

#### Schema Update

Add fields in `social_circles` to represent parent-created circles:

- **`created_by`** (Foreign Key to `parents`)
- **`is_custom`** (Boolean, to mark custom circles)

#### API Design

1. **Endpoint:** `POST /api/circles/create`
   - **Request:**
     ```json
     {
       "parent_id": 123,
       "circle_name": "Brigade Society Bus No. 92",
       "parent_circle_id": 1
     }
     ```

2. **Endpoint:** `GET /api/circles/discover`
   - **Response:**
     ```json
     [
       { "id": 1, "name": "Brigade Society Bus No. 92" },
       { "id": 2, "name": "Class II, DPS School" }
     ]
     ```

---

## Conclusion

This solution covers the end-to-end design of the data model and API for efficiently managing social circles, posts, replies, and schema evolution to accommodate future enhancements like child grade changes and parent-initiated circles.
C:\Users\lohar\readme.md