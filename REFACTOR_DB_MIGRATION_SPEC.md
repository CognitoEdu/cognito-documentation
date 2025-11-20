# Database Migration Specification

**Component**: SubtopicElements.jsx refactoring  
**Purpose**: Standardize activity allowance configurations to enable code removal  
**Date**: 2025-11-20

---

## Migration Overview

### Objective
Ensure ALL non-premium users have a standardized `activityAllowanceConfig` from the approved list, enabling removal of legacy fallback code.

### Scope
- Users with day-based allowance system
- Users without `activityAllowanceConfig`
- Users with `includeLessons: false` (if any)

### Expected Impact
- **Code reduction**: ~95 lines of legacy fallback code
- **Maintenance**: Simplified config logic (10 configs vs unbounded variations)
- **Performance**: No behavior change for users

---

## Step 1: Identify Affected Users

### Query 1: Day-Based Allowance Users

```javascript
// MongoDB query
db.users.find({
	'activityAllowance.currentFreeType': 'days',
	isPremiumAccount: { $ne: true },
	isActive: true,
	isArchived: { $ne: true }
}).count();
```

**Expected**: 0 users  
**If > 0**: Migration required

### Query 2: Users Without Config

```javascript
// MongoDB query
db.users.find({
	$or: [
		{ activityAllowanceConfig: { $exists: false } },
		{ activityAllowanceConfig: null }
	],
	isPremiumAccount: { $ne: true },
	isGuestAccount: { $ne: true },
	isActive: true,
	isArchived: { $ne: true }
}).count();
```

**Expected**: 0 users  
**If > 0**: Migration required

### Query 3: Users With includeLessons: false

```javascript
// MongoDB query
db.users.find({
	'activityAllowanceConfig.includeLessons': false,
	isPremiumAccount: { $ne: true },
	isActive: true,
	isArchived: { $ne: true }
}).count();
```

**Expected**: 0 users  
**If > 0**: Migration required

---

## Step 2: Migration Logic

### File Location
`common-server/scripts/migrate-activity-allowance-configs.js`

### Migration Rules

#### Rule 1: Day-Based → Element-Based

For users with `activityAllowance.currentFreeType === 'days'`:

```javascript
const migrateDayBasedUsers = async () => {
	const dayBasedUsers = await User.find({
		'activityAllowance.currentFreeType': 'days',
		isPremiumAccount: { $ne: true },
		isActive: true,
		isArchived: { $ne: true }
	});
	
	for (const user of dayBasedUsers) {
		// Determine appropriate cohort based on signup date
		const signupDate = user.dateTimeRegistered || user.dateTimeCreated;
		const daysSinceSignup = Math.floor(
			(Date.now() - signupDate.getTime()) / (1000 * 60 * 60 * 24)
		);
		
		let newConfig;
		
		if (daysSinceSignup <= 7) {
			// New users: generous trial
			newConfig = 'daily_75e_1a_15_with_lessons';
		} else if (daysSinceSignup <= 30) {
			// Trial period users
			newConfig = 'daily_75e_1a_10_with_lessons';
		} else {
			// Established users: sustainable ongoing
			newConfig = 'weekly_3a_with_lessons';
		}
		
		user.activityAllowanceConfig = newConfig;
		
		// Reset activity allowance to use element-based
		user.activityAllowance = {
			...user.activityAllowance,
			currentFreeType: 'elements',
			// Preserve existing usage data
		};
		
		await user.save();
		console.log(`Migrated user ${user.id} to ${newConfig}`);
	}
	
	console.log(`Migrated ${dayBasedUsers.length} day-based users`);
};
```

#### Rule 2: Missing Config → Default Assignment

For users without `activityAllowanceConfig`:

```javascript
const assignMissingConfigs = async () => {
	const usersWithoutConfig = await User.find({
		$or: [
			{ activityAllowanceConfig: { $exists: false } },
			{ activityAllowanceConfig: null }
		],
		isPremiumAccount: { $ne: true },
		isGuestAccount: { $ne: true },
		isActive: true,
		isArchived: { $ne: true }
	});
	
	for (const user of usersWithoutConfig) {
		const signupDate = user.dateTimeRegistered || user.dateTimeCreated;
		const daysSinceSignup = Math.floor(
			(Date.now() - signupDate.getTime()) / (1000 * 60 * 60 * 24)
		);
		
		let newConfig;
		
		// Conservative assignment for users without config
		if (daysSinceSignup <= 7) {
			// First week: generous trial
			newConfig = 'daily_75e_1a_15_with_lessons';
		} else if (daysSinceSignup <= 30) {
			// Trial month
			newConfig = 'daily_75e_1a_10_with_lessons';
		} else if (daysSinceSignup <= 90) {
			// Established users: balanced
			newConfig = 'weekly_5a_with_lessons';
		} else {
			// Long-term users: sustainable
			newConfig = 'weekly_3a_with_lessons';
		}
		
		user.activityAllowanceConfig = newConfig;
		
		// Initialize activity allowance if missing
		if (!user.activityAllowance) {
			user.activityAllowance = {
				initialFreeAllowance: 0,
				initialFreeType: 'elements',
				currentFreeAllowanceLessonElements: 0,
				currentFreeAllowanceQuizElements: 0,
				currentFreeAllowanceFlashcards: 0,
				currentFreeAllowanceExamQParts: 0,
				currentFreeAllowanceExamMCQs: 0,
				currentFreeAllowanceTotal: 0,
				currentFreeType: 'elements',
			};
		}
		
		await user.save();
		console.log(`Assigned config ${newConfig} to user ${user.id}`);
	}
	
	console.log(`Assigned configs to ${usersWithoutConfig.length} users`);
};
```

#### Rule 3: includeLessons: false → true

For users with `activityAllowanceConfig.includeLessons === false`:

```javascript
const migrateLessonsExcluded = async () => {
	const usersWithoutLessons = await User.find({
		'activityAllowanceConfig.includeLessons': false,
		isPremiumAccount: { $ne: true },
		isActive: true,
		isArchived: { $ne: true }
	});
	
	for (const user of usersWithoutLessons) {
		// Map to equivalent config with lessons
		const currentConfig = user.activityAllowanceConfig;
		
		let newConfig;
		
		if (currentConfig.period === 'daily') {
			newConfig = 'daily_75e_1a_10_with_lessons';
		} else {
			newConfig = 'weekly_5a_with_lessons';
		}
		
		user.activityAllowanceConfig = newConfig;
		await user.save();
		console.log(`Migrated user ${user.id} to include lessons`);
	}
	
	console.log(`Migrated ${usersWithoutLessons.length} users to include lessons`);
};
```

---

## Step 3: Verification Queries

### Pre-Migration Verification

Run these queries to determine scope:

```javascript
// 1. Count day-based users
const dayBasedCount = await User.countDocuments({
	'activityAllowance.currentFreeType': 'days',
	isPremiumAccount: { $ne: true },
	isActive: true
});
console.log(`Day-based users: ${dayBasedCount}`);

// 2. Count users without config
const noConfigCount = await User.countDocuments({
	$or: [
		{ activityAllowanceConfig: { $exists: false } },
		{ activityAllowanceConfig: null }
	],
	isPremiumAccount: { $ne: true },
	isGuestAccount: { $ne: true },
	isActive: true
});
console.log(`Users without config: ${noConfigCount}`);

// 3. Count users with lessons excluded
const noLessonsCount = await User.countDocuments({
	'activityAllowanceConfig.includeLessons': false,
	isPremiumAccount: { $ne: true },
	isActive: true
});
console.log(`Users with lessons excluded: ${noLessonsCount}`);

// 4. Total affected users
const totalAffected = dayBasedCount + noConfigCount + noLessonsCount;
console.log(`Total users requiring migration: ${totalAffected}`);
```

### Post-Migration Verification

Verify all non-premium users have valid configs:

```javascript
// All non-premium users should have one of these 10 configs
const validConfigs = [
	'guest_daily_0_1a_10_with_lessons',
	'daily_75e_1a_15_with_lessons',
	'daily_75e_1a_10_with_lessons',
	'daily_75e_1a_15_across_all_activities_with_lessons',
	'weekly_50e_25_with_lessons',
	'weekly_50e_50_across_all_activities_with_lessons',
	'weekly_7a_across_all_activities_with_lessons',
	'weekly_10a_across_all_activities_with_lessons',
	'weekly_3a_with_lessons',
	'weekly_5a_with_lessons'
];

const invalidConfigUsers = await User.find({
	isPremiumAccount: { $ne: true },
	isGuestAccount: { $ne: true },
	isActive: true,
	isArchived: { $ne: true },
	activityAllowanceConfig: { $nin: validConfigs }
});

console.log(`Users with invalid configs: ${invalidConfigUsers.length}`);
if (invalidConfigUsers.length > 0) {
	console.log('Invalid configs:', invalidConfigUsers.map(u => u.activityAllowanceConfig));
}
```

### Guest User Verification

```javascript
// Verify all guest users have guest config
const guestUsers = await User.find({
	isGuestAccount: true,
	isActive: true
});

const invalidGuestConfigs = guestUsers.filter(
	u => u.activityAllowanceConfig !== 'guest_daily_0_1a_10_with_lessons'
);

console.log(`Guest users with wrong config: ${invalidGuestConfigs.length}`);
```

---

## Step 4: Rollback Plan

### If Issues Detected

1. **Database backup** before running migration
2. **Keep audit log** of all user config changes
3. **Revert script** to restore original configs:

```javascript
// Restore from audit log
const rollbackMigration = async (auditLogPath) => {
	const auditLog = require(auditLogPath);
	
	for (const entry of auditLog) {
		const user = await User.findById(entry.userId);
		if (user) {
			user.activityAllowanceConfig = entry.originalConfig;
			user.activityAllowance = entry.originalAllowance;
			await user.save();
		}
	}
	
	console.log(`Rolled back ${auditLog.length} users`);
};
```

### Audit Log Format

```javascript
{
	timestamp: new Date(),
	userId: "user_id",
	originalConfig: "old_config_id",
	newConfig: "new_config_id",
	originalAllowance: { /* snapshot */ },
	newAllowance: { /* snapshot */ },
	migrationReason: "day-based-to-element-based"
}
```

---

## Step 5: Frontend Code Removal

### After Successful Migration

Once verification confirms all users have valid configs:

### Remove from SubtopicElements.jsx

**Lines 755**: 
```javascript
// REMOVE
const [freeDayAllowance, setFreeDayAllowance] = useState(false);
```

**Lines 1332-1369**: 
```javascript
// REMOVE entire else-if block
} else if (loginContext.activityAllowance.currentFreeType === 'days') {
	// ... 38 lines ...
}
```

**Lines 1396-1413**: 
```javascript
// REMOVE function
function getDifferenceInDays(date1, date2) {
	// ... 18 lines ...
}
```

**Line 1432**:
```javascript
// REMOVE check
if (freeDayAllowance) {
	usingFreeAllowance = true;
}
```

**Lines 1542-1550**:
```javascript
// REMOVE fallback
if (!loginContext.activityAllowanceConfig) {
	// ... 9 lines ...
}
```

**Line 1525**:
```javascript
// REMOVE fallback constant
let maxActivityAllowance = ActivityAccess.MAX_DAILY_ACTIVITY_ACCESS;
```

### Update Related Code

**Lines 1519-1574**: Simplify by removing fallback branch
- Keep: `else if (period === 'daily')` branch
- Keep: `else if (period === 'weekly')` branch
- Remove: Initial `if (!activityAllowanceConfig)` branch

**Lines 1428-1440**: Simplify allowance usage check
- Remove: `if (freeDayAllowance)` branch
- Keep: `if (freeElementAllowance > 0)` branch

---

## Step 6: Testing Plan

### Pre-Deployment Testing

**Test Cases**:
1. Guest user: Verify 1 activity, 10 elements daily limit
2. Cohort 1 user: 75 initial + 1 activity + 15 elements daily
3. Cohort 3 user: restrictAcrossAllActivities works correctly
4. Cohort 6 user: restrictByActivity with exam qs
5. Cohort 4 user: Weekly limits reset correctly
6. Resumed activities: Limits persist across sessions
7. Mixed subtopic activities: Correct subtopic tracking

### Smoke Tests
1. Start lesson (focus mode) - verify continues to work
2. Start quiz (all-at-once) - verify submission flow
3. Start flashcards - verify response handling
4. Hit daily/weekly limit - verify modal shows
5. Complete activity - verify XP awarded correctly

### Regression Tests
- Run full test suite
- Check activity limit modal behavior
- Verify analytics events still fire
- Test activity resumption

---

## Step 7: Deployment Strategy

### Phase 1: Migration Script (Backend)
1. Deploy migration script to production
2. Run verification queries
3. If affected users found: Run migration
4. Verify post-migration queries
5. Generate audit log

**Estimated time**: 5 minutes (assuming < 100 affected users)

### Phase 2: Frontend Code Removal
1. **After** migration verified successful
2. Deploy frontend with legacy code removed
3. Monitor for errors in first 24 hours
4. Keep rollback ready

**Estimated time**: Standard deployment cycle

### Rollback Criteria
- Any users unable to access activities
- Limit modals showing incorrectly
- Activity completion failures
- XP not awarded

---

## Communication Plan

### User Impact
**Expected**: ZERO user-facing changes  
**Actual behavior**: Identical before and after

### If Migration Changes User Limits
(Only if users were on unsupported configs)

**Email template**:
```
Subject: Your Cognito learning plan has been updated

Hi [FirstName],

We've updated your learning plan to give you a better experience on Cognito.

Your new plan:
- [X] activities per [day/week]
- [Y] questions per [day/week]
- Includes lessons, quizzes, flashcards, and exam questions

No action needed - just keep learning!

The Cognito Team
```

---

## Monitoring

### Post-Deployment Metrics

**Week 1 after deployment**:
- Monitor error rates in activity components
- Track activity completion rates
- Watch for limit-related support tickets
- Check analytics for anomalies

**Queries to run**:
```javascript
// Users who started activities
db.dailyActivity.find({
	dateAsString: TODAY,
	totalActivityCount: { $gt: 0 }
}).count();

// Users who hit limits
db.dailyActivity.find({
	dateAsString: TODAY,
	// Compare totalElementCount to user's limit
}).count();

// Activities completed
db.courseResults.find({
	dateTimeCompleted: { $gte: YESTERDAY }
}).count();
```

### Success Criteria

✅ **Migration successful if**:
- Zero users with invalid configs
- Activity completion rate unchanged (±2%)
- Error rate unchanged (±5%)
- Zero limit-related support tickets
- Analytics data consistent

❌ **Rollback if**:
- Activity completion rate drops > 10%
- Error rate increases > 20%
- Multiple limit-related support tickets
- Users reporting inability to access activities

---

## Timeline

### Week 1: Preparation
- Day 1: Run verification queries
- Day 2-3: Review results, adjust migration logic if needed
- Day 4: Dry run on staging environment
- Day 5: Review dry run results

### Week 2: Execution
- Day 1: Deploy migration script to production
- Day 1: Run migration (5-10 minutes)
- Day 1: Verify migration success
- Day 2-3: Monitor for issues
- Day 4: Deploy frontend code removal if no issues
- Day 5-7: Monitor post-deployment

### Week 3: Validation
- Review metrics
- Confirm code removal successful
- Archive legacy code for reference
- Update documentation

---

## Appendix: Current Cohort Configurations

### Daily Cohorts (Guest + 1-3)

**Guest**: 0 initial, 1 activity, 10 elements daily  
**Cohort 1**: 75 initial, 1 activity, 15 elements daily  
**Cohort 2**: 75 initial, 1 activity, 10 elements daily  
**Cohort 3**: 75 initial, 1 activity, 15 elements daily (across all activities)

### Weekly Cohorts (4-9)

**Cohort 4**: 50 initial, 25 elements weekly  
**Cohort 5**: 50 initial, 50 elements weekly (across all activities)  
**Cohort 6**: 0 initial, 7 activities weekly (across all, by activity)  
**Cohort 7**: 10 initial, 10 activities weekly (across all, by activity)  
**Cohort 8**: 0 initial, 3 activities weekly (by activity)  
**Cohort 9**: 0 initial, 5 activities weekly (by activity)

### Configuration Flags

**ALL configs have**:
- `includeLessons: true` ✓
- `initialFreeType: 'elements'` or `initialFreeAllowance: 0` ✓
- NOT `currentFreeType: 'days'` ✓

**Configuration variations in use**:
- `period`: daily (4 configs) | weekly (6 configs)
- `restrictAcrossAllActivities`: true (4 configs) | false (6 configs)
- `restrictByActivity`: true (4 configs) | false (5 configs) | undefined (1 config)

---

## Sign-Off

### Pre-Migration Checklist

- [ ] Verification queries run
- [ ] Affected user count documented
- [ ] Migration script tested on staging
- [ ] Rollback plan documented
- [ ] Monitoring plan established
- [ ] Communication templates ready (if needed)

### Post-Migration Checklist

- [ ] All users have valid configs
- [ ] Zero errors in migration log
- [ ] Audit log generated
- [ ] Verification queries confirm success
- [ ] 24-hour monitoring complete
- [ ] Frontend code removal approved

### Code Removal Checklist

- [ ] Legacy branches removed (95 lines)
- [ ] Tests updated
- [ ] No references to removed code
- [ ] Documentation updated
- [ ] Code review approved
- [ ] Deployed to production
- [ ] Post-deployment monitoring complete

---

## Approval

**Backend Migration**: _______________ (Tech Lead) Date: _______

**Frontend Removal**: _______________ (Tech Lead) Date: _______

**Deployment**: _______________ (DevOps) Date: _______

