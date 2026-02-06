# Note Taking and Report Writing
## Notes
We should record exactly what we do. This includes:
- Every command we type
- lines of code we modify
- any gui clicks (if necessary)
Need to be structured and sufficiently detailed, so we can recreate steps at a later date
Choose a good tool --> obsidian
use screenshots to supplement notes, not replace them
- show only one concept per screenshot
- frame important info properly
- caption
print screen key should capture entire screen in Kali Linux and save to images directory
shift+printscreen should allow for highlighting
screenshot tool also available

## Reports
The audience may influence the level of detail and severity of finding --> need to know audience and impact
We should be including high level info that management can understand, while also including enough technical detail that technical staff are able to understand report and implement remediation
### Executive Summary
Should be first section of the report
Outline:
- Scope
- What was tested / did scope change
- timeframe of test
	- length of test
	- dates
- rules of engagement
	- verify they were followed
- Supporting infrastructure and accounts
	- Accounts created
	- network connections
	- attacking IPs
	- etc
Long form executive summary
written summary of testing that provides high level overview
- Major findings
- Any trends during testing --> are findings related in any way
- Anything done well by client
- quick wrap up --> "more details in the following sections"
### Test Environment Considerations
Document any circumstances or limitations during testing
usually small section
### Technical summary
List all key findings in the report, written out with summary and recommendations
group findings into common areas
Finish section with risk heat map based on client context
### Technical Findings and Recommendation
Section where we will include full technical details
Broad background on why something can take place is usually good
Finding description should be a sentence or two describing what the vulnerability is, why it is dangerous, and what the impact is.
Then we describe some technical details on the vulnerability
then describe specific finding --> command / screen shots
remediation
