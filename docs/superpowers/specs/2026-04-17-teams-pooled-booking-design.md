# Teams with Pooled Booking - Design Specification

**Date:** 2026-04-17
**Status:** Draft - Awaiting Approval

---

## Context

Currently, every user in seatsurfing can book up to 3 seats per day for themselves. This design introduces **Teams** - a feature that allows users to pool their booking capacity so that designated team leaders can book seats for the entire team. Team-owned bookings are available for any member to use on a first-come-first-served basis.

This is intentionally separate from the existing **Groups** feature, which is used for access control and booking approvals.

---

## Requirements

### User Stories

1. **As a user**, I want to create a team so that I can coordinate seat booking with my colleagues.
2. **As a team member**, I want to set how many of my daily bookings to contribute to the team pool.
3. **As a team member**, I want to be able to book seats for the team using the team's booking contingency.
4. **As a team member**, I want to see team-owned bookings on my calendar so I know which seats are available for me to use.
5. **As an org admin**, I want to be able to delete any team for governance purposes.

### Functional Requirements

1. **Team Creation:** Any user can create a team.
2. **Team Membership:** Users can add/remove themselves from teams. Team creator (leader) can manage memberships.
3. **Contribution Limits:** Each member contributes as many bookings per day from their daily capacity to the team as they wish. They can define this in the team UI.
4. **Team Bookings:** Bookings made from the pool are owned by the team, not individuals.
5. **Calendar Display:** Team bookings appear on all members' calendars with distinct styling.
6. **Deletion:** Only org admins can delete teams.

---

## Data Model

### New Tables

```sql
-- Teams table
CREATE TABLE teams (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name varchar(255) NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT unique_team_name UNIQUE (organization_id, name)
);

-- Team membership with contribution limits and leadership flag
CREATE TABLE team_members (
    team_id uuid NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    contribution_limit smallint NOT NULL DEFAULT 0 CHECK (contribution_limit >= 0 AND contribution_limit <= 3),
    is_leader boolean NOT NULL DEFAULT false,
    joined_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);

-- Bookings made from team pool (team-owned, not individual-owned)
CREATE TABLE team_bookings (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id uuid NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    space_id uuid NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    enter timestamptz NOT NULL,
    leave timestamptz NOT NULL,
    booked_by uuid NOT NULL REFERENCES users(id), -- leader who made the booking
    created_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT valid_time_range CHECK (leave > enter)
);

-- Indexes for performance
CREATE INDEX idx_team_bookings_team_date ON team_bookings(team_id, enter);
CREATE INDEX idx_team_bookings_space_date ON team_bookings(space_id, enter, leave);
CREATE INDEX idx_team_members_user ON team_members(user_id);
```

### Relationships

```
Organization (1) -> (N) Team (N) <- (1) Team_Member (N) <- (1) User
                            |
                            |-> (N) Team_Booking
```

---

## API Design

### Teams

| Method | Endpoint | Permissions | Description |
|--------|----------|-------------|-------------|
| GET | `/team/` | Authenticated | List all teams user is member of |
| POST | `/team/` | Authenticated | Create a new team |
| GET | `/team/{id}` | Member or Leader | Get team details |
| PUT | `/team/{id}` | Leader | Update team name |
| DELETE | `/team/{id}` | Org Admin | Delete team |

### Team Members

| Method | Endpoint | Permissions | Description |
|--------|----------|-------------|-------------|
| GET | `/team/{id}/member` | Member | List team members |
| POST | `/team/{id}/member` | Leader | Add member to team |
| PUT | `/team/{id}/member/{userId}` | Leader or Self | Update contribution/role |
| DELETE | `/team/{id}/member/{userId}` | Leader or Self | Remove member from team |

### Team Bookings

| Method | Endpoint | Permissions | Description |
|--------|----------|-------------|-------------|
| GET | `/team/{id}/booking` | Member | List team bookings |
| POST | `/team/{id}/booking` | Member | Create team booking |
| DELETE | `/team/{id}/booking/{id}` | Member | Cancel team booking |

### Team Pool Status

| Method | Endpoint | Permissions | Description |
|--------|----------|-------------|-------------|
| GET | `/team/{id}/pool/status` | Member | Get pool capacity and usage for a date |

---

## Backend Implementation

### Repository Pattern

Create `team-repository.go` following existing patterns:

```go
type TeamRepository struct{}

// Team operations
Create(t *Team) error
GetOne(id string) (*Team, error)
GetAllForUser(userID string) ([]*Team, error)
GetAllForOrganization(orgID string) ([]*Team, error)
Update(t *Team) error
Delete(t *Team) error

// Member operations
GetMembers(teamID string) ([]*TeamMember, error)
AddMember(teamID string, userID string, contributionLimit int, isLeader bool) error
UpdateMember(teamID string, userID string, contributionLimit int, isLeader bool) error
RemoveMember(teamID string, userID string) error

// Booking operations
CreateBooking(b *TeamBooking) error
GetBookingsForTeam(teamID string, date time.Time) ([]*TeamBooking, error)
GetPoolUsage(teamID string, date time.Time) (int, error) // returns bookings count for date
DeleteBooking(id string) error
```

### Router

Create `team-router.go` with RESTful endpoints as specified above.

### Validation Logic

Key validation rules in booking creation:

```go
func validateTeamBooking(teamID string, leaderID string, date time.Time) error {
    // 1. Verify user is a leader of the team
    // 2. Calculate total pool capacity (sum of member contributions)
    // 3. Count existing team bookings for the date
    // 4. Ensure bookings < pool capacity
    // 5. Check space availability (no conflicts)
}

func validatePersonalBooking(userID string, date time.Time) error {
    // 1. Get user's team contributions
    // 2. Calculate remaining personal limit (3 - total_contributions)
    // 3. Count existing personal bookings for the date
    // 4. Ensure bookings <= remaining limit
}
```

---

## Frontend Implementation

### New Pages

1. **`/ui/src/pages/teams/index.tsx`** - Team list and creation
   - List all teams user is member of
   - "Create Team" button
   - Quick stats (pool size, today's usage)

2. **`/ui/src/pages/teams/[id].tsx`** - Team management
   - Team name edit
   - Member list with contribution controls
   - Leader toggle for each member
   - Add/remove members
   - Pool status display

### Navigation

Add "Teams" tab to top navigation bar (`/ui/src/components/TopNav.tsx` or equivalent):

```
[Book Seat] [My Bookings] [My Colleagues] [Teams] [Settings] [Logout]
```

### Booking UI Changes

In the main booking flow, add a toggle for leaders:

```
┌─────────────────────────────────────────┐
│ □ Book for Team: [Select Team ▼]       │
└─────────────────────────────────────────┘
```

When enabled:
- Team selector appears
- Booking is created as team-owned
- Validation uses pool capacity instead of personal limit

### Calendar Display

Team bookings should display with distinct styling:

```typescript
// In calendar cell rendering
const getBookingStyle = (booking: Booking) => {
    if (booking.isTeamBooking) {
        return { backgroundColor: '#e3f2fd', borderLeft: '4px solid #2196f3' };
    }
    return { backgroundColor: '#f5f5f5' };
};
```

---

## Types

### TypeScript (`/ui/src/types/Team.ts`)

```typescript
export class Team extends Entity {
    id: string;
    organizationId: string;
    name: string;
    createdAt: string;

    getBackendUrl(): string { return "/team/"; }
    async save(): Promise<Team>
    async delete(): Promise<void>
    async getMembers(): Promise<TeamMember[]>
    async addMember(userId: string, contributionLimit: number, isLeader: boolean): Promise<void>
    async updateMember(userId: string, contributionLimit: number, isLeader: boolean): Promise<void>
    async removeMember(userId: string): Promise<void>
    async getPoolStatus(date: Date): Promise<PoolStatus>
}

export class TeamMember extends Entity {
    teamId: string;
    userId: string;
    user: User; // populated from API
    contributionLimit: number;
    isLeader: boolean;
    joinedAt: string;
}

export interface PoolStatus {
    date: string;
    totalCapacity: number;
    used: number;
    available: number;
}

export interface TeamBooking extends Entity {
    teamId: string;
    spaceId: string;
    enter: string;
    leave: string;
    bookedBy: string;
    bookedByUser: User; // populated from API
}
```

---

## Feature Flag

Add new feature flag `feature_teams` in settings:

```go
const SettingFeatureTeams = "feature_teams"
```

Default: `false` (disabled by default, enabled per-organization)

---

## Migration Plan

1. **Schema Migration** (version 40):
   - Add `teams` table
   - Add `team_members` table
   - Add `team_bookings` table
   - Add indexes

2. **Backend**:
   - Implement `team-repository.go`
   - Implement `team-router.go`
   - Add validation logic to booking router
   - Add feature flag checks

3. **Frontend**:
   - Add TypeScript types
   - Create Team API client
   - Build Teams pages
   - Update navigation
   - Modify booking flow
   - Update calendar rendering

4. **Testing**:
   - Unit tests for repository
   - API tests for router
   - E2E tests for team creation, membership, booking
   - Validation tests for pool limits

---

## Edge Cases & Considerations

1. **Leader leaves team:** If the last leader leaves, automatically promote a member or require admin intervention.

2. **Contribution changes mid-day:** Changes to contribution limits take effect the next day. Current day's pool is locked.

3. **Team deletion:** All team bookings must be cancelled or transferred before deletion. If bookings still exists, deletion will fail in back-end and report error to front-end.

4. **Space restrictions:** Team bookings still respect space-level allowed bookers (groups feature).

5. **Timezone handling:** Pool usage is calculated per calendar day in the organization's timezone.

6. **Recurring bookings:** Team bookings do not support recurring patterns (out of scope for v1).

7. **Team Leader leaving:** If the last team leader wants to leave a team, it fails. They have to promote another member to leader before leaving.

8. **Single Team:** A user can only be part of a single team.

---

## Verification

### Manual Testing Checklist

- [ ] Create a team as a regular user
- [ ] Add members to team
- [ ] Set contribution limits
- [ ] Member books from team pool
- [ ] Verify pool capacity enforcement
- [ ] Verify personal limit reduced by contribution
- [ ] Team bookings appear on member calendars
- [ ] Team bookings have distinct styling
- [ ] Non-admin cannot delete team
- [ ] Admin can delete any team

### API Tests

- [ ] Team CRUD operations
- [ ] Member management
- [ ] Pool status calculation
- [ ] Booking validation (pool limits)
- [ ] Permission enforcement

---

## Open Questions and Decisions

1. Should teams have a description field? -> out of scope
2. Should there be a team avatar/logo? -> out of scope
3. Should pool status show per-location breakdown? -> out of scope
4. Should members be notified when a team booking is made? -> out of scope
5. Space-level restrictions: Space-level restrictions for teams are calculated as the intersection between all members.
6. Concurrency: The back-end ensures that Team bookings are synchronized. In case of race-conditions (two people booking the last spot simultaneously) the back-end will throw an error for the second booking.

---

## Next Steps

1. Review and approve this design
2. Create implementation plan using `superpowers:writing-plans` skill
3. Execute implementation
4. Test and deploy
