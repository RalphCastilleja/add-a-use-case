# **Student account lifecycle**

**Realizes** [UC-STU-deactivate-student](../requirements/use-cases.md#uc-stu-deactivate-student-the-course-admininstructor-deactivates-a-student)  
**Depends on** FR-SEC-active-account, BR-student-lifecycle, BR-submission-owed, BR-inactive-peer-evaluations

This flow uses the existing [platform containers](architectural-design.md) and student account model. The existing enabled flag is sufficient. The design requires no schema change.

```mermaid
sequenceDiagram
    actor Instructor as Course admin or instructor
    participant Details as Student details
    participant Security as SecurityConfiguration
    participant Controller as StudentController
    participant Service as StudentService
    participant Repository as StudentRepository
    Instructor->>Details: Choose Deactivate
    Details-->>Instructor: Show consequences and ask for confirmation
    Instructor->>Details: Confirm
    Details->>Security: POST /students/{studentId}/deactivate
    Security->>Controller: Authorized role
    Controller->>Service: Deactivate selected student
    Service->>Repository: Check scope and update enabled flag
    Repository-->>Service: Updated
    Controller-->>Details: 204 No Content
    Details-->>Instructor: Refresh details and show success
```

## **Account updates and access**

- Register the POST matcher under the configured API base in `SecurityConfiguration` for `ROLE_admin` and `ROLE_instructor`. Resolve instructor scope from the target student's current course section in the service. Check scope before disclosing account status. Use 403 for denied scope, 404 for a missing student, and 409 for an already inactive student.
- Apply the state change transactionally. Make the enabled-to-disabled transition conditional so two confirmations cannot both succeed. Remove the student status dropdown and the generic update path's ability to write the enabled flag.
- On authenticated student requests, resolve the current account using the stable student identity in the JWT and reject inactive accounts at authentication with 401. Preserve existing token validation and login checks.

## **Performance tracking**

- Reuse `SubmissionStatusService` for reminder and owed-item eligibility. Keep the team roster intact and filter the peer evaluation form's targets independently of the general team roster.
- In `EvaluationService`, enforce eligibility on submission even if the form was loaded earlier. For weekly reporting, include an inactive student only if a saved evaluation for that student exists in the requested week. Preserve the existing behavior for active students.

## **Test list**

| Scenario | Source | Expected result |
| --- | --- | --- |
| Confirm deactivation | Steps 3–6 | Warning covers consequences. Confirm sends the dedicated request and shows success after the server updates the account. |
| Cancel warning | 4a | Details remain visible and the client sends no mutation. |
| Preserve records and team | BR-student-lifecycle | Student is inactive and the team, saved reports, and saved evaluations are retained. |
| Recompute owed work | BR-submission-owed | Withdrawn student owes neither item and an active teammate can finish without rating her. |
| Reject inactive evaluation target | BR-inactive-peer-evaluations | Form omits the target and a stale submission is rejected without adding a rating. |
| Keep historical reporting | BR-inactive-peer-evaluations | Inactive student appears with saved ratings for the requested week and is absent for a week without them. |
| Deny existing credentials | 6a, FR-SEC-active-account | A token that worked before deactivation is denied on the next protected request. A new login is denied too. |
| Repeat deactivation | 3a | A second confirmation reports the student already inactive and preserves state. |
| Enforce actor scope | 3b, BR-student-lifecycle | Course admin and assigned instructor can deactivate. Unassigned instructor and student receive a denial with no state change. |
| Prevent generic edit bypass | Steps 3–6 | Generic profile updates cannot change account status. |
