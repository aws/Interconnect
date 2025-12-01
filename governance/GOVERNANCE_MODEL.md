# Governance Model - Project Governance Framework

This open source project is managed by a Steering Committee composed of representatives from Maintaining Organizations. Maintaining Organizations are defined as the Founding Members and any additional members added by way of the processes described herein, listed in the MAINTAINING_ORGS.md file in the project repository. 

## Member Categories

The project recognizes three categories of members:
1. **Founding Members**: Amazon Web Services and Google are designated as Founding Members of this project
2. **Maintaining Organizations**: Organizations who contribute to and maintain the specification, with full commit access to the project repositories
3. **Implementers**: Organizations that are actively implementing the specification in accordance with the Security and Operational Excellence Policy in production cloud environments, but have not yet become a Maintaining Organization
Addition and Removal of Members
Founding Members

The Founding Members are the initial Maintaining Organizations. Removal of a Founding Member requires consent of that founding member. Aside from removal, Founding Members are treated the same as any other member and have no special voting or governance privileges beyond removal protection and those of its members who serve on the Steering Committee.
Maintaining Organizations

To qualify as a Maintaining Organization, an organization must demonstrate:
1.	Commitment to the Security and Operational Excellence Policy, demonstrated by 2 and 3;
2.	Proven ability to implement and maintain the specification in a production environment for at least a year; and
3.	History of constructive, security-compliant contribution to the project.
The Steering Committee will evaluate Maintaining Organization candidates to determine whether they have met the above criteria, with particular emphasis on their demonstrated commitment to the project's security and operational standards. Maintaining Organizations may be added or removed by no less than 3/4 affirmative vote of the Steering Committee, however a Maintaining Organization is free to remove itself at any time by written notice to the Steering Committee. 
Implementers

Implementers, listed in the IMPLEMENTERS.md file in the repository, may be added or removed by no less than 3/4 affirmative vote of the Steering Committee, based on an assessment of whether the production implementation aligns with all requirements of the specification and the Security and Operational Excellence Policy.
Removal Criteria

A Maintaining Organization or Implementer may be considered for removal by the Steering Committee if any of the following conditions are met:

1.	Lack of Participation: The member has been inactive or non-responsive in project activities for a period of six months or more, including failure to participate in votes, respond to communications, or contribute to the project's development.
2.	Policy Non-Compliance: The member has demonstrated a pattern of non-compliance with the Security and Operational Excellence Policy.
3.	Disruptive Conduct: The member has engaged in conduct that significantly disrupts the project's operations or damages its reputation, including violations of the project Code of Conduct https://aws.github.io/code-of-conduct.
Any Maintaining Organization may propose another member for removal by submitting a written request to the Steering Committee Chair, who will then initiate the removal process. The member proposed for removal will be given reasonable notice and an opportunity to address the concerns before a vote is taken.

Removal requires a 3/4 affirmative vote of the Steering Committee, with the member proposed for removal recusing themselves from the vote if they are a Maintaining Organization, except for Founding Members who must consent to their removal. Any member may also choose to remove themselves.
Representation 

The Steering Committee will be responsible for oversight of all technical, project, approval, and policy matters for the project. Each Maintaining Organization will designate one Steering Committee member, listed in the STEERING_COMMITTEE.md file in the repository. This representative will cast the Maintaining Organization's single vote in formal voting matters. 

The Steering Committee will appoint a Chair responsible for organizing the Steering Committee’s activity. If the Steering Committee Chair is removed from the Committee (or the Chair steps down from that role), it is the responsibility of the Steering Committee to appoint a new Chair.

Each Maintaining Organization may designate up to 3 individuals with full commit access to the project repositories, listed in the MAINTAINERS.md file in the repository, and can provide as many experts as needed for the Technical Advisory Board (TAB).

Maintaining Organizations have full voting rights through participation in the Steering Committee, and Implementers have voting rights on changes that impose significant implementation burdens, with each Implementer getting a single vote.
Voting

The Steering Committee will strive for all decisions to be made by consensus. While explicit agreement of the entire Steering Committee is preferred, it is not required for consensus. Rather, the Steering Committee will determine consensus based on their good faith consideration of a number of factors, including the collective views of the Steering Committee members, the technical merits of the positions expressed, alignment with project goals, and the substantive nature of support and objections. 

If consensus cannot be reached, the Steering Committee will make the decision by a vote. The Steering Committee Chair will call a vote with reasonable notice to the Steering Committee, setting out a discussion period and a separate voting period.  

Voting will also be used for:
•	the addition or removal of Maintaining Organizations and Implementers;
•	adoption of resource-intensive changes; 
•	changes to this Project Governance framework; and 
•	changes to the Security and Operational Excellence Policy.
Except as specifically noted elsewhere in this document, decisions by vote require a simple majority vote of all voting members.
Resource-Intensive Changes

For changes that would impose significant implementation burdens (substantial hardware changes, major architectural modifications, or significant engineering effort), the following special voting process applies:
1.	Any Maintaining Organization may designate a proposed change as "resource-intensive" during the discussion period.
2.	If designated as resource-intensive, the change requires a vote including the Steering Committee and a representative from each Implementer.
To be approved, a resource-intensive change must receive a majority vote from the Maintaining Organizations and at least one vote from an Implementer, if there are Implementers. This ensures that no single party can unilaterally impose significant resource requirements on other implementers, while still allowing the specification to evolve.
Security and Operational Excellence Policy Changes

Changes to the Security and Operational Excellence Policy require a 3/4 affirmative vote of the Steering Committee, reflecting the critical importance of maintaining high security standards.
Technical Advisory Board

A Technical Advisory Board (TAB) consisting of technical experts from Maintaining Organizations will convene as needed to:
1.	Evaluate the technical feasibility of proposed changes
2.	Evaluate compliance of proposed changes with the Security and Operational Excellence Policy
3.	Assess implementation burden of proposed changes, and provide recommendations to the Steering Committee on resource-intensive changes
The TAB will operate in an advisory capacity and does not have direct voting rights beyond those of its members who serve on the Steering Committee.
Transparency

All Steering Committee and TAB meetings, decisions, and recommendations shall be documented and published in the project repository, subject to reasonable restrictions on confidentiality to maintain commercially sensitive information or information pertaining to the security of the project or the Maintaining Organizations.
Annual Review

The Steering Committee will conduct an annual review of all of the following:
1.	This Project Governance 
2.	The Security and Operational Excellence Policy
3.	Maintaining Organization composition and contributions
4.	Implementer membership
