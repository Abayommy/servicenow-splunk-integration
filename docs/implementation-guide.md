# ServiceNow to Splunk Implementation Guide

## Pre-Implementation Checklist

### ServiceNow Environment
- [ ] ServiceNow instance accessible
- [ ] API user account created
- [ ] Required permissions assigned:
  - [ ] `rest_service`
  - [ ] `web_service_admin` 
  - [ ] `itil`
  - [ ] Read access to incident table
- [ ] Test incidents created for validation

### Splunk Environment  
- [ ] Splunk Enterprise installed
- [ ] ServiceNow Add-on installed
- [ ] Admin access available
- [ ] Network connectivity confirmed

## Detailed Implementation Steps

### Phase 1: Environment Preparation (30 minutes)

#### AWS Infrastructure Setup
```bash
# Connect to Splunk instance
ssh -i ~/path/to/keypair.pem ec2-user@[SPLUNK_IP]

# Switch to splunk user
sudo su - splunk

# Verify Splunk service status
/opt/splunk/bin/splunk status
ServiceNow Add-on Verification
bash# List installed apps
/opt/splunk/bin/splunk list app

# Look for: Splunk_TA_snow - Status should show installed
# If not found, install from Splunkbase
Phase 2: ServiceNow Configuration (45 minutes)
Step 1: Access ServiceNow Instance

Navigate to your ServiceNow developer instance
Log in with admin credentials
Verify API access is enabled

Step 2: Create Test Data
javascript// In ServiceNow, create test incidents via System Definition > Scripts - Background
var inc = new GlideRecord('incident');
inc.initialize();
inc.short_description = 'Test incident for Splunk integration';
inc.description = 'Network connectivity issues - test data';
inc.state = 1; // New
inc.priority = 3; // Moderate
inc.urgency = 1; // High
inc.impact = 2; // Medium
inc.caller_id = gs.getUserID();
inc.assignment_group = 'Network';
inc.category = 'Network';
inc.insert();
gs.print('Created incident: ' + inc.number);
Phase 3: Splunk Add-on Configuration (60 minutes)
Step 1: Access Splunk Web Interface

Open browser: http://[SPLUNK_IP]:8000
Login with Splunk admin credentials
Navigate: Apps → Splunk Add-on for ServiceNow

Step 2: Account Configuration

Click Configuration tab
Click Add in ServiceNow Account section
Fill in details:

Account Name: ServiceNow_Production (or descriptive name)
ServiceNow URL: https://your-instance.service-now.com
Username: Your ServiceNow API user
Password: Your ServiceNow API user password
Authentication Type: Basic Authentication
Record Count: 3000



Step 3: Test Connection

Click Test Connection button
Verify success message appears
Save configuration

Step 4: Data Input Configuration

Go to Inputs tab
Click Create New Input
Configure:

Name: servicenow_incidents_prod
ServiceNow Account: Select account created above
Collection Interval: 300 (5 minutes)
ServiceNow Table: incident
Time Field: sys_updated_on
Start Date: Leave default (one week ago)
Record Count: 3000
Include Fields:
number,short_description,description,state,priority,urgency,impact,
assigned_to,assignment_group,caller_id,category,subcategory,
opened_at,resolved_at,closed_at,sys_created_on,sys_updated_on

Filter Parameters: active=true
Unique ID Field: sys_id
Index: main (or dedicated ServiceNow index)


Enable the input
Click Save

Phase 4: Validation & Testing (30 minutes)
Step 1: Monitor Data Collection
splunk# Check for data ingestion (wait 5-10 minutes after configuration)
index=main sourcetype="snow:incident"
| head 10
Step 2: Verify Field Extraction
splunk# Validate field extraction
index=main sourcetype="snow:incident"
| table _time, dv_number, short_description, dv_state, dv_priority, dv_urgency
| head 5
Step 3: Collection Health Check
splunk# Monitor collection logs
index=_internal "*servicenow*" OR "*snow*" OR "*ServiceNow*"
| head 20
Step 4: Validate Data Completeness
splunk# Count total incidents collected
index=main sourcetype="snow:incident"
| stats count by dv_state
Phase 5: Production Optimization (45 minutes)
Index Optimization
bash# Create dedicated index (optional)
# In Splunk Web: Settings > Indexes > New Index
# Index Name: servicenow
# Update data input to use this index
Field Extraction Enhancement
splunk# Create calculated fields for better analysis
# Settings > Fields > Calculated Fields
# Add field: incident_age
# Expression: round((now() - strptime(dv_opened_at, "%Y-%m-%d %H:%M:%S"))/86400, 0)
Dashboard Creation

Create new dashboard: ServiceNow Operations
Add panels:

Incident volume by priority (timechart)
Top assignment groups (stats)
Average resolution time (eval)
Critical incidents (table)



Phase 6: Monitoring Setup (30 minutes)
Health Monitoring Search
splunk# Save as scheduled search: "ServiceNow Collection Health"
index=_internal source=*servicenow* "Data collection completed"
| rex field=_raw "Got a total of (?<record_count>\d+) records"
| eval collection_time=strftime(_time, "%Y-%m-%d %H:%M:%S")
| table collection_time, record_count
| sort -_time
Error Alerting
splunk# Save as alert: "ServiceNow Collection Errors"
index=_internal source=*servicenow* (ERROR OR WARN)
| table _time, message
# Alert conditions: Number of results > 0
# Email notification to ops team
Troubleshooting Common Issues
No Data Appearing

Check ServiceNow connectivity:
bashcurl -u username:password "https://instance.service-now.com/api/now/table/incident?sysparm_limit=1"

Verify add-on logs:
bashtail -f /opt/splunk/var/log/splunk/splunkd.log | grep -i servicenow

Validate user permissions in ServiceNow

Authentication Failures

Password expired: Reset ServiceNow user password
Account locked: Unlock in ServiceNow user management
Wrong permissions: Add required roles to API user

Performance Issues

Reduce collection interval if overwhelming ServiceNow
Limit fields collected to essential data only
Add filters to reduce record volume

Production Deployment Checklist

 Dedicated ServiceNow index created
 Production credentials configured
 Collection interval optimized for environment
 Monitoring searches scheduled
 Error alerting configured
 Documentation updated with production details
 Backup of configuration exported
 Team training completed

Next Steps

Expand data collection: Add problems, changes, CMDB
Create advanced dashboards: Executive KPIs, SLA tracking
Implement machine learning: Incident classification, prediction
Integrate with SOAR: Automated response workflows


Estimated Total Implementation Time: 4-5 hours
Skills Required: Splunk Administration, ServiceNow API, Basic scripting

