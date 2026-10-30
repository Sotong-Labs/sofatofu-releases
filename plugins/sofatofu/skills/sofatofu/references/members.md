# Members and invitations

Members are the people who can access the workspace. They're not CRM person records.

Every member can list the members. Only owners can invite someone, remove a member, list invitations or revoke an invitation.

These writes are recorded as changes, but `change reverse` can't undo them. To correct access, take another action instead, such as revoking an invitation or sending a new one.

## List members and invitations

```text
member list {"limit":100}
invitation list {"limit":100}
```

`member list` returns active members, and `invitation list` returns pending invitations. Set `email` in `invitation list` to see only the pending invitations for one address. The match ignores case and surrounding spaces.

A member item has `id`, `name`, `email` and `role`. The `id` is the member's user ID. Use it for the `owner` key in `records` and `patch`, and for the `author` filter of `change list`.

An invitation item has `id`, `email`, `role`, `expires_at` and `invitation_link`.

## Invite someone

First read the pending invitations for the email you plan to invite:

```text
invitation list {"email":"colleague@example.com"}
```

If `total` is 0, no pending invitation uses that email, so invite with that read's `read_id`:

```text
member invite {"email":"colleague@example.com","reason":"Give the new sales colleague workspace access","read_id":"<read_id>"}
```

The invite adds the person with the `member` role. The result has the invitation's ID in `ids`, plus `delivery` details.

Check `delivery.sent`. If email delivery failed, the invitation still exists. Include `delivery.link` in your task result and say that the invitation exists but the email didn't send. See [Talking to the user](../SKILL.md#talking-to-the-user). Don't create another invitation, and don't wait for a reply.

An invitation can be accepted for 48 hours. The invited person has to sign in with the invited email address and accept it in the browser. Once it expires, you can invite the person again.

Your part of the task is done once you've created the invitation and reported delivery. Wait for acceptance only if the task needs it.

### If the invite is refused

- **`invalid_input` saying the email already belongs to a member or pending invitation.** Check the workspace's members and pending invitations. An empty invitation read doesn't prove the person isn't a member.
- **Wrong read.** The read has to be for the email you're inviting. A read for a different email, or an incomplete page, returns `read_scope`.
- **A pending invitation turned up.** Don't invite again.
- **`read_stale`.** Repeat the same `email` read and check the result again.

## Remove a member

Use the `id` and `read_id` from a `member list` page that contains the member:

```text
member remove {"id":"9Xq4tLm2Rv7Kp1Nz8Bw3Hc6Jd0Fy5Gsa","reason":"Member left the company","read_id":"<read_id>"}
```

The member loses access right away. A `sole_owner` error means the workspace has only one owner, so that membership can't be removed.

## Revoke an invitation

Use an invitation ID and `read_id` from an `invitation list` page that contains the invitation:

```text
invitation revoke {"id":"c4d8e2f6g0h3","reason":"Invitation used the wrong email address","read_id":"<read_id>"}
```

Revoking cancels a pending invitation. If the person has already joined, remove the member instead.
