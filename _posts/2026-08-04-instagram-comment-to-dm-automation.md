---
layout: post
title: "Comment LINK, Get a DM: Automating Instagram's Private Replies"
date: 2026-08-04
tags: [instagram, webhooks, automation, api, golang]
---

Instagram captions aren't clickable, and a business account only gets one link, in its bio. If every post needs a different URL, the common workaround is the "comment LINK and I'll send it to you" pattern. You've almost certainly seen it, usually run by a human typing it out one comment at a time. It doesn't have to be a person doing that. Instagram's own APIs cover the whole loop: a webhook tells you someone commented, and a private-reply endpoint lets you message them back referencing that exact comment.

## The shape of the problem

Say you run an Instagram business account where every post should point somewhere different: a product, an article, a listing, whatever. Three things have to happen, in order, entirely server-side:

1. Get told, in near-real-time, when someone comments on a specific post.
2. Figure out what that post is actually supposed to link to.
3. Send that person a DM containing the link, without them having to follow you or message first.

Step 2 is on you. Instagram's webhook payload gives you a comment and the ID of the media it's on, nothing about what that media means to your business. You need your own mapping from media ID back to a URL, built at post time. Steps 1 and 3 are Meta's Instagram Platform APIs, and that's the part worth covering here.

## Step 1: Subscribe to comment events

A webhook subscription is per-field. You tell Instagram which fields to notify you about via the `subscribed_apps` endpoint:

```bash
curl -i -X POST \
  "https://graph.instagram.com/v25.0/<IG_USER_ID>/subscribed_apps?subscribed_fields=comments,messages&access_token=<ACCESS_TOKEN>"
```

`comments` is what gets you notified when someone comments on a post. Subscribe to `messages` too even if you're not handling inbound DMs yet. It's easy to forget, and then wonder later why a related feature silently gets no events.

## Step 2: Handle the verification handshake

Before Instagram sends real events, it sends one `GET` request to prove you own the endpoint. You get a `hub.verify_token` you chose when configuring the webhook in the App Dashboard, and you just have to echo back the challenge if it matches:

```go
func handleWebhookVerify(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query()
	if q.Get("hub.mode") == "subscribe" && q.Get("hub.verify_token") == webhookVerifyToken() {
		w.Write([]byte(q.Get("hub.challenge")))
		return
	}
	w.WriteHeader(http.StatusForbidden)
}
```

## Step 3: Verify every real event is actually from Meta

Once subscribed, every webhook delivery arrives as a `POST` with an `X-Hub-Signature-256` header: an HMAC-SHA256 of the raw request body, keyed with your app secret. Your webhook URL has to be public for Meta to reach it, which means anyone who finds it can `POST` a fake comment event unless you check this:

```go
func verifyMetaSignature(body []byte, signatureHeader string) bool {
	const prefix = "sha256="
	if !strings.HasPrefix(signatureHeader, prefix) {
		return false
	}
	expected, err := hex.DecodeString(strings.TrimPrefix(signatureHeader, prefix))
	if err != nil {
		return false
	}
	mac := hmac.New(sha256.New, []byte(metaAppSecret()))
	mac.Write(body)
	return hmac.Equal(expected, mac.Sum(nil))
}
```

`hmac.Equal` instead of `bytes.Equal` matters here. A naive byte comparison leaks timing information about how many leading bytes matched, a real (if narrow) side channel for forging a signature. Reject anything that doesn't verify before you even look at the payload.

## Step 4: Parse the comment and decide whether to reply

The payload shape for a comment event nests a few levels deep:

```go
type commentWebhookPayload struct {
	Entry []struct {
		Changes []struct {
			Field string `json:"field"`
			Value struct {
				ID   string `json:"id"` // comment ID
				Text string `json:"text"`
				From struct {
					ID string `json:"id"` // commenter's Instagram-scoped ID
				} `json:"from"`
				Media struct {
					ID string `json:"id"`
				} `json:"media"`
			} `json:"value"`
		} `json:"changes"`
	} `json:"entry"`
}
```

From here it's plain application logic: check `Field == "comments"`, check the comment text matches your trigger word, look up `Media.ID` in whatever mapping you built at post time, and bail out early (acking the webhook with `200` regardless, since Meta retries on anything else) if there's no match.

Build in a per-commenter cooldown from the start. Nothing stops someone from commenting your trigger word five times in a row, and without a cooldown you'll send five private replies for it. A simple in-memory `map[string]time.Time` keyed by the commenter's ID, checked against a cooldown window, is enough if your webhook handler runs as a single instance. A redeploy just resets everyone's cooldown, which is harmless.

## Step 5: Send the private reply

This is the actual DM. It's a `POST` to the Instagram user's own `/messages` endpoint, addressed by `comment_id` instead of a recipient ID you'd otherwise have to already know:

```bash
curl -i -X POST "https://graph.instagram.com/v25.0/<IG_USER_ID>/messages" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -d '{
        "recipient": {"comment_id": "<COMMENT_ID>"},
        "message": {"text": "<YOUR REPLY TEXT>"}
      }'
```

A success response hands back `recipient_id` and `message_id`:

```json
{
  "recipient_id": "526...",
  "message_id": "aWdfZ..."
}
```

In Go, that's a small, self-contained function:

```go
func sendPrivateReply(commentID, message string) error {
	body, _ := json.Marshal(map[string]any{
		"recipient": map[string]string{"comment_id": commentID},
		"message":   map[string]string{"text": message},
	})
	req, err := http.NewRequest(http.MethodPost,
		"https://graph.instagram.com/v25.0/"+igUserID+"/messages",
		bytes.NewReader(body))
	if err != nil {
		return err
	}
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Authorization", "Bearer "+igAccessToken())

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		respBody, _ := io.ReadAll(resp.Body)
		return fmt.Errorf("private reply failed (%d): %s", resp.StatusCode, respBody)
	}
	return nil
}
```

## The limitations that actually matter

A few constraints are easy to miss until they bite in production:

- **7-day window.** You can only send the initial private reply within 7 days of the comment, or during a live broadcast. Comment on a month-old post and this silently won't work.
- **Top-level comments only.** You can't target a reply to a reply. Instagram folds those into the top-level comment automatically, so that's what you're actually replying against.
- **Live videos are excluded** from this specific flow; Meta's docs point at the general Messaging API for that case instead.
- **One shot, then it's their move.** You get exactly one private reply per comment. A follow-up message only works if the recipient responds first, and then only within 24 hours of that response. This isn't a channel for an ongoing conversation. It's a single nudge.

None of these are things you can code around. They're policy, not a bug in your integration, so design for them instead of finding out the hard way during an incident.

## References

- [Instagram Platform: Private Replies](https://developers.facebook.com/documentation/instagram-platform/private-replies)
- [Instagram Platform: Webhooks](https://developers.facebook.com/documentation/instagram-platform/webhooks)
- [IG Comment: Replies reference (limitations)](https://developers.facebook.com/documentation/instagram-platform/instagram-graph-api/reference/ig-comment/replies)
