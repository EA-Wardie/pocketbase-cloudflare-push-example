## PocketBase Vloudflare Push Notifications Example
A simple implementation example for push notifications using PocketBase and Cloudflare workers.

#### Prerequisites
- PWA or web app with a service worker.
- PocketBase instance.
- 3rd party service to host a worker function (e.g. [Cloudflare](https://workers.cloudflare.com/)).

#### PocketBase

##### Create `subsctions` and `notifications` collections respectively
- Subscriptions should have a column of type `JSON` (e.g. `data`).
- Notifications can have any columns, but push messages require at least a title (e.g. `id`, `title`, `body`, `created`, `updated`).

**Extending PocketBase with JavaScript**

Listen for notification created events and respond by calling your worker function.

**pb_hooks/main.pb.js**
```JS
onRecordAfterCreateSuccess((e) => {
	e.next();
	
	try {
		$http.send({url: `<url-to-worker-function>?id=${e.record.get('id')}`});
	} catch (error) {
		// handle error
	}
}, "notifications");
```

Notifications can be manually created from your admin view, or create notifications on creation events for other records (.e.g. `posts`).

#### VAPID Keys

You can generate your public and private VAPID keys using the `web-push` library.

See docs for [web-push](https://github.com/web-push-libs/web-push?tab=readme-ov-file#command-line).

#### PWA / Web App

##### Subscribe to push notifications using the Web Push API

**main.ts**
```JS
const registration: ServiceWorkerRegistration = await navigator.serviceWorker.ready; // Your active service worker registration
const permission: NotificationPermission = await Notification.requestPermission(); // This will prompt users with a popup

// Only create or update subscriptions if permissions as granted
if (permission === 'granted') {
	// Retreive the currect subsction for this browser if it exists
	registration.pushManager.getSubscription().then(async (existingSubscription: PushSubscription | null) => {
		// If no currect subscription, subscribe with Web Push API and save to PocketBase
		if (!existingSubscription) {
			registration.pushManager.subscribe({
				userVisibleOnly: true,
				applicationServerKey: urlBase64ToUint8Array(PUBLIC_VAPID_KEY), // Your public VAPID key
			}).then(async (subscription) => {
				await pb.table('subscriptions').create({
					data: subscription,
				});
			});
		}

		// If a subscription exists, query collection based on endpoint
		// If no subscription exists on PocketBase, create one
		if (existingSubscription) {
			await pb.table('subscriptions').getFirstListItem(`data.endpoint="${existingSubscription.endpoint}"`).catch(async (error) => {
				if(error.status === 404) {
					await pb.table('subscriptions').create({
						data: existingSubscription,
					});
				}
			);
		}
	});
}
```

Although prompting users on site load works fine for Chromium based browsers, it does not on Safari. For Safari this code must be run on user interaction (e.g. tapping a button).

##### Example for a base 64 to unit 8 array helper function

**helpers.js**
```JS
// This is LLM code, so test before going to production
export function urlBase64ToUint8Array(base64String: string) {
	const padding = '='.repeat((4 - (base64String.length % 4)) % 4);
	const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
	const rawData = atob(base64);
	const outputArray = new Uint8Array(rawData.length);

	for (let i = 0; i < rawData.length; ++i) {
		outputArray[i] = rawData.charCodeAt(i);
	}
	
	return outputArray;
}
```

##### Listen for push events in your service worker

**service-worker.js**
```JS
self.addEventListener('push', (event: PushEvent) => {
	const data: Record<string, any> = event.data?.json();
	const title: string = data.title || '';
	const options = {
		body: data.body,
		// icon: '/icon.png',
		// badge: '/badge.png',
	};

	if (title) {
		event.waitUntil(self.registration.showNotification(title, options)); // This creates the actual notification on the device
	}
});
```

#### Worker / Edge Function

The code here is depended on your worker / edge function of choice.
This example is for **Cloudflare** workers.

```JS
import { buildPushPayload, type PushMessage, type PushSubscription, type VapidKeys } from '@block65/webcrypto-web-push';

export default {
	async fetch(request, env): Promise<Response> {
		let subscriptions: PushSubscription[] = [];

		// Query all subscriptions form your PocketBase instance
		// The "env.API_URL" here points to your PocketBase api url
		try {
			const subscriptionsResponse = await fetch(`${env.API_URL}/collections/subscriptions/records?skipTotal=true&fields=data`);
			const subscriptionsJson = await subscriptionsResponse.json<{ items: { data: PushSubscription }[] }>();

			subscriptions = subscriptionsJson.items.map(({ data }) => data);
		} catch (error) {
			subscriptions = [];

			return new Response('failed to fetch subscribers', { status: 500 });
		}

		const url = new URL(request.url);
		 // To keep things simple you can add the notification id to your request url query params
		 // This can also be done with headers or posting JSON to your worker e.g.
		const notificationId = url.searchParams.get('id');

		if (!notificationId) {
			return new Response('no notification id provided in query params', { status: 404 });
		}

		let notification: Record<string, any> | undefined;

		// Query the notification you want to send from PocketBase
		try {
			const notificationResponse = await fetch(`${env.API_URL}/collections/notifications/records/${notificationId}`);

			notification = await notificationResponse.json();
		} catch (error) {
			notification = undefined;

			return new Response('failed to fetch notification', { status: 500 });
		}


		if (!notification) {
			return new Response('no notification found', { status: 404 });
		}

		// Build a list requests for each subscriber
		const requestPromises = subscriptions.map(async (subscription) => {
			// Basic push message
			const message: PushMessage = {
				data: notification,
				options: {
					urgency: 'normal'
				}
			};

			// The same keys you generated earlier
			const vapidKeys: VapidKeys = {
				subject: 'mailto:mail@example.com',
				publicKey: env.VAPID_SERVER_PUBLIC_KEY,
				privateKey: env.VAPID_SERVER_PRIVATE_KEY
			};

			// Build a push payload for each subscriber
			try {
				const payload = await buildPushPayload(message, subscription, vapidKeys);

				return { subscription, payload };
			} catch (error) {
				// buildPushPayload() will fail if your message, subscription or VAPID keys are invalid
				return undefined;
			}
		});

		const requests = await Promise.all(requestPromises);

		// Create a list of promises with each payload
		// Filter out invalid promises
		const promises: Promise<Response>[] = requests
			.filter((request) => !!request?.payload)
			// The "fetch()" here is what queues your push notification with the subscriber's push serive
			.map((request) => fetch(request?.subscription.endpoint || '', request?.payload));

		// Send all push notifications
		// They will queue with their respective push services depending on urgency
		// Devices might start receiving notifications immediately, or after a few minutes
		const responses = await Promise.all(promises);

		// Basic response and error handling
		const printList = responses.map((response) => `${response.url}: ${response.status}`);

		return new Response(JSON.stringify(printList), { status: 200 });
	}
} satisfies ExportedHandler<Env>;
```

PS. The code in these examples are a simple example of sending push notifications and improvements are almost certainly possible.
