

```typescript
// https://community.cloudflare.com/t/timeout-with-fetch/25249/5

export async function readRequestBody(request: Request) {
	try {
		if (request.method !== 'GET') {
			let contentType = request.headers.get('content-type');

			if (includes(contentType, 'application/json')) {
				return await request.json();
			} else if (includes(contentType, 'form')) {
				let formData = await request.formData();
				let body = {};
				formData.forEach((value, key) => {
					body[key] = value;
				});
				return { text: body };
			} else {
				return { text: request.text() };
			}
		}
	} catch (e) {
		console.error('readRequestBody: ', e);
	}
	return { text: '' };
}

export function postBearer(data: object, apikey: string): any {
	return {
		headers: {
			Authorization: `Bearer ${apikey}`,
			'Content-Type': 'application/json',
			'Accept': 'application/json',
		},
		method: 'POST',
		body: JSON.stringify(data),
	};
}

export const JSON_HEADER = {
	'content-type': 'application/json;charset=UTF-8',
};
export const HTML_HEADER = {
	'content-type': 'text/html;charset=UTF-8',
};

export const NOT_FOUND = () => new Response('404 Not Found', { status: 404 });
export const UNPROCESSABLE_ENTITY = () => new Response('422 Unprocessable Content', { status: 422 });
export const INTERNAL_SERVER_ERROR = () => new Response('500 Internal Server Error', { status: 500 });

export function includesArr(arr: [], item) {
	for (let i = 0; i < arr.length; i++) {
		if (includes(arr[i], item)) {
			return true;
		}
	}
	return false;
}

export function includes(str: string, item) {
	return str === null ? false : str.indexOf(item) !== -1;
}

```
