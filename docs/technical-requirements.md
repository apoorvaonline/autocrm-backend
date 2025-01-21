# AutoCRM Technical Requirements

## Database Schema

### Core Tables

#### users

- `id`: uuid (primary key)
- `email`: text (unique)
- `full_name`: text
- `role`: enum ('customer', 'employee', 'admin')
- `created_at`: timestamp
- `last_login`: timestamp
- `is_active`: boolean
- `preferences`: jsonb (notification settings, UI preferences)

#### teams

- `id`: uuid (primary key)
- `name`: text
- `description`: text
- `created_at`: timestamp
- `updated_at`: timestamp
- `is_active`: boolean

#### team_members

- `id`: uuid (primary key)
- `team_id`: uuid (foreign key to teams)
- `user_id`: uuid (foreign key to users)
- `role`: enum ('member', 'lead')
- `joined_at`: timestamp

#### tickets

- `id`: uuid (primary key)
- `customer_id`: uuid (foreign key to users)
- `assigned_to`: uuid (foreign key to users, nullable)
- `team_id`: uuid (foreign key to teams, nullable)
- `status`: enum ('new', 'open', 'pending', 'resolved', 'closed')
- `priority`: enum ('low', 'medium', 'high', 'urgent')
- `category`: text
- `subject`: text
- `description`: text
- `source`: enum ('email', 'chat', 'web', 'sms')
- `created_at`: timestamp
- `updated_at`: timestamp
- `resolved_at`: timestamp
- `sla_breach_at`: timestamp
- `metadata`: jsonb (for flexible additional data)

#### ticket_messages

- `id`: uuid (primary key)
- `ticket_id`: uuid (foreign key to tickets)
- `sender_id`: uuid (foreign key to users)
- `message_type`: enum ('reply', 'note', 'system')
- `content`: text
- `sentiment_score`: float (nullable)
- `created_at`: timestamp
- `attachments`: jsonb (array of attachment metadata)

#### ticket_activities

- `id`: uuid (primary key)
- `ticket_id`: uuid (foreign key to tickets)
- `user_id`: uuid (foreign key to users)
- `activity_type`: enum ('status_change', 'assignment', 'comment', 'priority_change')
- `details`: jsonb
- `created_at`: timestamp

#### knowledge_base_articles

- `id`: uuid (primary key)
- `title`: text
- `content`: text
- `category`: text
- `author_id`: uuid (foreign key to users)
- `status`: enum ('draft', 'published', 'archived')
- `created_at`: timestamp
- `updated_at`: timestamp
- `published_at`: timestamp
- `views_count`: integer

### Supporting Tables

#### ticket_tags

- `id`: uuid (primary key)
- `ticket_id`: uuid (foreign key to tickets)
- `tag`: text
- `created_at`: timestamp

#### canned_responses

- `id`: uuid (primary key)
- `title`: text
- `content`: text
- `created_by`: uuid (foreign key to users)
- `team_id`: uuid (foreign key to teams, nullable)
- `created_at`: timestamp
- `updated_at`: timestamp

#### notifications

- `id`: uuid (primary key)
- `user_id`: uuid (foreign key to users)
- `type`: enum ('ticket_update', 'mention', 'assignment', 'sla_breach')
- `content`: jsonb
- `read`: boolean
- `created_at`: timestamp

### Indexes

- users(email)
- tickets(customer_id, created_at)
- tickets(assigned_to, status)
- tickets(team_id, status)
- ticket_messages(ticket_id, created_at)
- ticket_activities(ticket_id, created_at)
- notifications(user_id, read, created_at)

### Real-time Subscriptions

- tickets (for live updates)
- ticket_messages (for live chat)
- notifications (for instant notifications)

### Row Level Security (RLS) Policies

1. Users can only read their own profile
2. Customers can only view and update their own tickets
3. Employees can view tickets assigned to them or their team
4. Admins have full access to all tables
5. Public access is denied by default

### Notes

- Using UUID for primary keys to support distributed systems
- JSONB fields for flexible metadata storage
- Timestamp fields include timezone information
- Enums used for fixed-choice fields to ensure data consistency
