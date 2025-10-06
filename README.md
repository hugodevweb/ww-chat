# WW-CHAT

A modern, feature-rich chat component for WeWeb with support for mentions, attachments, real-time messaging, and customizable styling.

## Features

-   💬 **Real-time messaging** with automatic scrolling
-   👥 **@Mentions** with dropdown selection and custom highlighting
-   📎 **File attachments** with preview support
-   🎨 **Fully customizable** styling (colors, fonts, borders, etc.)
-   👤 **User avatars** and status indicators
-   📅 **Date separators** for message grouping
-   🔔 **Event triggers** for message sent, attachment clicks, etc.
-   🌍 **Localization** support for dates and times
-   ♿ **Accessible** with keyboard navigation

## Installation

To run locally, first install all dependencies with `npm install`.

## Development

To serve locally, run `npm run serve --port=[PORT]`, and then go to WeWeb editor, open developer popup and add `localhost:[PORT]` as custom element.

## Build

Before release, you can check build errors by running `npm run build --name="ww-chat" --type="element"`

---

## 📋 Message Data Structure

### Basic Message Format

Messages passed to the component should follow this structure:

```javascript
{
  id: "msg-123",                           // Unique message ID
  text: "@John Doe can you check this?",  // Message text with mentions
  senderId: "user-456",                    // ID of message sender
  userName: "Alice",                       // Display name of sender
  timestamp: "2025-10-06T10:30:00Z",      // ISO timestamp
  attachments: [...],                      // Optional: array of attachments
  mentions: [                              // Optional: array of mentioned users
    {
      id: "user-789",
      name: "John Doe",
      avatar: "https://..."
    }
  ]
}
```

### Message Mapping

Configure how the component reads your message data:

-   **Message ID**: `context.mapping['id']`
-   **Message Text**: `context.mapping['text']`
-   **Sender ID**: `context.mapping['senderId']`
-   **Timestamp**: `context.mapping['timestamp']`
-   **Attachments**: `context.mapping['attachments']` (optional)

---

## 👥 Mentions System

The component includes a fully-featured mentions system that allows users to mention other participants using the `@` symbol.

### How Mentions Work

1. **User types `@`** in the input → Mentions dropdown appears
2. **User selects a participant** → `@Name` is inserted into the message
3. **Message is sent** → The `mentions` array is included in the message data
4. **Message is displayed** → Mentions are automatically highlighted with custom colors

### Storing Mentions in Your Database

#### Recommended Database Schema

**Option 1: JSON Column (Recommended)**

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY,
  text TEXT NOT NULL,
  sender_id UUID NOT NULL,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  mentions JSONB DEFAULT '[]'::jsonb,  -- Store mentions as JSON

  FOREIGN KEY (sender_id) REFERENCES users(id)
);
```

**Example Database Row:**

```json
{
    "id": "msg-123",
    "text": "@John Doe hey, can you check this with @Jane Smith?",
    "sender_id": "user-456",
    "timestamp": "2025-10-06T10:30:00Z",
    "mentions": [
        {
            "id": "user-789",
            "name": "John Doe",
            "avatar": "https://example.com/john.jpg"
        },
        {
            "id": "user-012",
            "name": "Jane Smith",
            "avatar": "https://example.com/jane.jpg"
        }
    ]
}
```

**Option 2: Separate Mentions Table (Normalized)**

```sql
-- Messages table
CREATE TABLE messages (
  id UUID PRIMARY KEY,
  text TEXT NOT NULL,
  sender_id UUID NOT NULL,
  timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- Mentions table (many-to-many)
CREATE TABLE message_mentions (
  id UUID PRIMARY KEY,
  message_id UUID NOT NULL,
  user_id UUID NOT NULL,

  FOREIGN KEY (message_id) REFERENCES messages(id) ON DELETE CASCADE,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### WeWeb Workflow: Saving Messages with Mentions

**On Message Sent Event:**

```javascript
// Trigger: Chat component "On message sent"
// The event provides: trigger.event.message

// Save to database
const { data, error } = await supabase
    .from('messages')
    .insert({
        text: trigger.event.message.text,
        sender_id: trigger.event.message.senderId,
        timestamp: trigger.event.message.timestamp,
        mentions: trigger.event.message.mentions || [], // Store mentions array
    })
    .select()
    .single();

// Optional: Send notifications to mentioned users
if (trigger.event.message.mentions?.length > 0) {
    await supabase.from('notifications').insert(
        trigger.event.message.mentions.map(mention => ({
            user_id: mention.id,
            type: 'mention',
            message_id: data.id,
            sender_id: trigger.event.message.senderId,
        }))
    );
}
```

### WeWeb Workflow: Loading Messages with Mentions

**Query with Mentions:**

```javascript
// Load messages from database
const { data: messages } = await supabase
    .from('messages')
    .select(
        `
    id,
    text,
    sender_id,
    timestamp,
    mentions,  -- Include mentions in the query
    sender:users!sender_id (
      name,
      avatar
    )
  `
    )
    .eq('conversation_id', conversationId)
    .order('timestamp', { ascending: true });

// Format for the chat component
return messages.map(msg => ({
    id: msg.id,
    text: msg.text,
    senderId: msg.sender_id,
    userName: msg.sender.name,
    timestamp: msg.timestamp,
    mentions: msg.mentions, // Component uses this for highlighting!
}));
```

### Configuring Mention Highlights

In the WeWeb editor, customize mention appearance:

-   **Mentions Color**: Text color for highlighted mentions (default: `#3b82f6`)
-   **Mentions Background**: Background color for mentions (default: `#dbeafe`)

The component automatically:

-   ✅ Finds mentions in the message text
-   ✅ Highlights them with your custom colors
-   ✅ Handles multiple mentions correctly
-   ✅ Works with adjacent mentions (e.g., `@John @Jane`)

### Mentions Data Flow

```
User types @
    ↓
Dropdown shows participants
    ↓
User selects participant
    ↓
"@Name" inserted in text
    ↓
Message sent with mentions array
    ↓
Saved to database with mentions
    ↓
Loaded back from database
    ↓
Component highlights mentions automatically
```

---

## 📎 Attachments

### Attachment Data Structure

```javascript
{
  id: "file-1",
  name: "document.pdf",
  type: "application/pdf",  // MIME type
  size: 1024000,            // Size in bytes
  url: "https://..."        // URL to the file
}
```

### Handling Attachments

**On Message Sent (with attachments):**

```javascript
// The event provides File objects in trigger.event.message.attachments
const files = trigger.event.message.attachments;

// Upload files to storage (e.g., Supabase Storage)
const uploadedFiles = [];
for (const file of files) {
    const { data, error } = await supabase.storage
        .from('chat-attachments')
        .upload(`${conversationId}/${file.name}`, file);

    if (!error) {
        const url = supabase.storage.from('chat-attachments').getPublicUrl(data.path).data.publicUrl;

        uploadedFiles.push({
            id: crypto.randomUUID(),
            name: file.name,
            type: file.type,
            size: file.size,
            url: url,
        });
    }
}

// Save message with attachment URLs
await supabase.from('messages').insert({
    text: trigger.event.message.text,
    sender_id: trigger.event.message.senderId,
    timestamp: trigger.event.message.timestamp,
    attachments: uploadedFiles,
});
```

---

## 🎨 Customization

The component offers extensive styling options:

### Header

-   Background color, text color, border
-   Name and location font sizes
-   Close button styling
-   Group chat avatar color

### Messages

-   Background colors for own/other messages
-   Text colors, font sizes, font weights
-   Border and border radius
-   Custom message bubbles

### Input Area

-   Background color, text color
-   Border styles (normal, hover, focus)
-   Placeholder styling
-   Button colors and sizes

### Icons

-   Send icon, attachment icon, remove icon
-   Customizable colors and sizes
-   Use any Lucide icon

---

## 🔔 Events

The component emits the following events:

### messageSent

Triggered when a user sends a message.

```javascript
{
  message: {
    id: "msg-123",
    text: "Hello @John!",
    senderId: "user-456",
    userName: "Alice",
    timestamp: "2025-10-06T10:30:00Z",
    attachments: [...],  // File objects
    mentions: [...]      // Array of mentioned users
  }
}
```

### attachmentClick

Triggered when a user clicks an attachment in a message.

```javascript
{
  attachment: {
    id: "file-1",
    name: "document.pdf",
    type: "application/pdf",
    size: 1024000,
    url: "https://..."
  }
}
```

### messageRightClick

Triggered when a user right-clicks a message.

```javascript
{
  message: { /* message object */ },
  position: {
    elementX: 50,    // Relative to chat element
    elementY: 20,
    viewportX: 320,  // Relative to page
    viewportY: 480
  }
}
```

### close

Triggered when the user clicks the header close button.

---

## 🌍 Localization

Configure date/time formatting:

-   **Locale**: Choose from 40+ locales (e.g., `enUS`, `fr`, `de`, `ja`)
-   **Time Format**: Customize using date-fns format patterns (e.g., `h:mm a`, `HH:mm`)
-   **Custom Text**: Set custom text for "Today", "Yesterday", "Just now"

---

## 💡 Pro Tips

1. **Always include the `mentions` array** in your database schema, even if empty
2. **Don't extract mentions with regex** - use the array provided by the component
3. **Index the mentions column** (GIN index in PostgreSQL) for fast queries
4. **Send notifications** when users are mentioned
5. **Keep mention names at time of mention** for historical accuracy
6. **Use conversation-based queries** to load only relevant messages
7. **Enable auto-scroll** for better UX in real-time chats

---

## 📚 Additional Resources

-   [WeWeb Documentation](https://docs.weweb.io/)
-   [date-fns Format Patterns](https://date-fns.org/docs/format)
-   [Lucide Icons](https://lucide.dev/)

---

## License

MIT License - See LICENSE file for details
