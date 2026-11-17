# Space membership migration

Space membership now lives in DynamoDB. Neptune keeps space metadata and
knowledge. The normal installer update or `scripts/deploy-terraform.sh` performs
the migration; there is no separate migration command.

Existing space Admins become Owners. Platform Admin remains a separate role and
does not grant space membership. See [space roles](../using-the-platform/projects.md#space-roles)
for current access rules.

## Installation and update

Avoid membership changes and space creation/deletion during the update. Old
Lambdas write Neptune while new ones use DynamoDB, so changes during this window
can be missed. **If deployment partially fails, re-apply promptly** to finish
switching the application over.

Terraform performs the following steps:

1. Create the access and audit tables.
2. Run `membership_copy`, converting graph memberships and recording an audit
   baseline in one transaction per member. Verify the copy and index propagation
   before deploying application Lambdas.
3. Deploy the Lambdas with DynamoDB access.
4. Run the read-only `membership_report` to identify differences left by changes
   during deployment. The report logs failures without blocking the update.

Finish the frontend deployment and reload browser tabs.

The copy runs once. A failed copy is safe to retry: existing destination records
and audit baselines prevent overwriting grants or restoring removed members.
A successful copy is not repeated on later updates, including retries of a
partially completed application deployment. A large copy may exceed the Lambda's
15-minute limit and need another update attempt.

The copy merges duplicate memberships using the strongest valid grant, skips
spaces with no members, and warns about spaces with no Owner. It never invents
an Owner. Invalid identities, roles, or verification failures block deployment;
inspect the migration Lambda logs for the affected spaces.

## Check the result

Review `membership_report` and its logs after the update:

| Count                | Meaning                                         |
| -------------------- | ----------------------------------------------- |
| `missingDestination` | A graph membership is missing from DynamoDB     |
| `missingSource`      | A DynamoDB membership is missing from the graph |
| `roleMismatches`     | The two stores disagree on the role             |

Compare differences with membership history before correcting access. Graph
edges stop being updated after cutover, so a difference can be a legitimate new
grant or removal. Never copy an old graph grant back without checking.

If `reported: false`, fix the reported error and rerun the report. Adjust the
function name and AWS region/profile for your deployment:

```bash
aws lambda invoke \
  --function-name collaborative-ai-dlc-migrate-memberships-dev \
  --cli-binary-format raw-in-base64-out \
  --payload '{"action":"report"}' \
  membership-report.json
```

## API compatibility

Project and member lists still return arrays for older clients. Clients sending
`paged=1` receive `{ items, nextToken }`; follow the token even after an empty page.
A newly added space can take time to appear in the list, but protected operations
check the current membership directly.

Legacy `admin` mutation inputs are accepted as `owner`. Current roles are Owner,
Member, and Viewer. Membership history is available through
`GET /projects/{projectId}/membership-history` with `from`, `to`, and `nextToken`;
there is no history screen yet. See the [data model](../concepts/data-model.md#dynamodb-space-access-and-audit)
for storage details.

## Rollback

Retain both DynamoDB tables and the old graph edges. Before application cutover,
the graph is still authoritative. After cutover, rolling back to graph-based
code can lose new grants or restore revoked access. Reconcile changes first;
a Terraform state backup does not back up membership data.

Restore the matching frontend when rolling back. New browser bundles accept
legacy list responses until users reload.
