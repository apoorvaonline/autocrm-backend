## SQL Scripts

### 1. Create Enums

```sql
-- Create custom types for enums
CREATE TYPE user_role AS ENUM ('customer', 'employee', 'admin');
CREATE TYPE team_member_role AS ENUM ('member', 'lead');
CREATE TYPE ticket_status AS ENUM ('new', 'open', 'pending', 'resolved', 'closed');
CREATE TYPE ticket_priority AS ENUM ('low', 'medium', 'high', 'urgent');
CREATE TYPE ticket_source AS ENUM ('email', 'chat', 'web', 'sms');
CREATE TYPE message_type AS ENUM ('reply', 'note', 'system');
CREATE TYPE activity_type AS ENUM ('status_change', 'assignment', 'comment', 'priority_change');
CREATE TYPE article_status AS ENUM ('draft', 'published', 'archived');
CREATE TYPE notification_type AS ENUM ('ticket_update', 'mention', 'assignment', 'sla_breach');
```

### 2. Create Core Tables

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    full_name TEXT NOT NULL,
    role user_role NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_login TIMESTAMPTZ,
    is_active BOOLEAN DEFAULT true,
    preferences JSONB DEFAULT '{}'::jsonb
);

-- Teams table
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    is_active BOOLEAN DEFAULT true
);

-- Team members table
CREATE TABLE team_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id UUID REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role team_member_role NOT NULL,
    joined_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(team_id, user_id)
);

-- Tickets table
CREATE TABLE tickets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    customer_id UUID REFERENCES users(id) ON DELETE CASCADE,
    assigned_to UUID REFERENCES users(id) ON DELETE SET NULL,
    team_id UUID REFERENCES teams(id) ON DELETE SET NULL,
    status ticket_status NOT NULL DEFAULT 'new',
    priority ticket_priority NOT NULL DEFAULT 'medium',
    category TEXT,
    subject TEXT NOT NULL,
    description TEXT,
    source ticket_source NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    resolved_at TIMESTAMPTZ,
    sla_breach_at TIMESTAMPTZ,
    metadata JSONB DEFAULT '{}'::jsonb
);
```

### 3. Create Supporting Tables

```sql
-- Ticket messages table
CREATE TABLE ticket_messages (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ticket_id UUID REFERENCES tickets(id) ON DELETE CASCADE,
    sender_id UUID REFERENCES users(id) ON DELETE SET NULL,
    message_type message_type NOT NULL,
    content TEXT NOT NULL,
    sentiment_score FLOAT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    attachments JSONB DEFAULT '[]'::jsonb
);

-- Ticket activities table
CREATE TABLE ticket_activities (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ticket_id UUID REFERENCES tickets(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    activity_type activity_type NOT NULL,
    details JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Knowledge base articles table
CREATE TABLE knowledge_base_articles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    category TEXT,
    author_id UUID REFERENCES users(id) ON DELETE SET NULL,
    status article_status NOT NULL DEFAULT 'draft',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    published_at TIMESTAMPTZ,
    views_count INTEGER DEFAULT 0
);

-- Ticket tags table
CREATE TABLE ticket_tags (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ticket_id UUID REFERENCES tickets(id) ON DELETE CASCADE,
    tag TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(ticket_id, tag)
);

-- Canned responses table
CREATE TABLE canned_responses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    created_by UUID REFERENCES users(id) ON DELETE SET NULL,
    team_id UUID REFERENCES teams(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Notifications table
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    type notification_type NOT NULL,
    content JSONB NOT NULL DEFAULT '{}'::jsonb,
    read BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 4. Create Indexes

```sql
-- Create indexes for better query performance
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_tickets_customer ON tickets(customer_id, created_at);
CREATE INDEX idx_tickets_assigned ON tickets(assigned_to, status);
CREATE INDEX idx_tickets_team ON tickets(team_id, status);
CREATE INDEX idx_ticket_messages ON ticket_messages(ticket_id, created_at);
CREATE INDEX idx_ticket_activities ON ticket_activities(ticket_id, created_at);
CREATE INDEX idx_notifications ON notifications(user_id, read, created_at);
```

### 5. Enable Row Level Security

```sql
-- Enable RLS on all tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE teams ENABLE ROW LEVEL SECURITY;
ALTER TABLE team_members ENABLE ROW LEVEL SECURITY;
ALTER TABLE tickets ENABLE ROW LEVEL SECURITY;
ALTER TABLE ticket_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE ticket_activities ENABLE ROW LEVEL SECURITY;
ALTER TABLE knowledge_base_articles ENABLE ROW LEVEL SECURITY;
ALTER TABLE ticket_tags ENABLE ROW LEVEL SECURITY;
ALTER TABLE canned_responses ENABLE ROW LEVEL SECURITY;
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

-- Create policies (example for tickets table)
CREATE POLICY "Customers can view their own tickets"
ON tickets FOR SELECT
TO authenticated
USING (
    auth.uid() = customer_id OR
    auth.uid() = assigned_to OR
    EXISTS (
        SELECT 1 FROM users
        WHERE users.id = auth.uid()
        AND role = 'admin'
    ) OR
    EXISTS (
        SELECT 1 FROM team_members
        WHERE team_members.user_id = auth.uid()
        AND team_members.team_id = tickets.team_id
    )
);
```

### 6. Create Triggers for updated_at

```sql
-- First, create a function that will be used by all triggers
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Create triggers for all tables with updated_at column
CREATE TRIGGER update_teams_updated_at
    BEFORE UPDATE ON teams
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_tickets_updated_at
    BEFORE UPDATE ON tickets
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_knowledge_base_articles_updated_at
    BEFORE UPDATE ON knowledge_base_articles
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_canned_responses_updated_at
    BEFORE UPDATE ON canned_responses
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```
