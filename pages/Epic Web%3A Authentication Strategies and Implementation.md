- {{renderer :tocgen2}}
- # Cookies
  collapsed:: true
	- ## Theme cookie
		- used `cookie` pnpm package
			- to parse the `theme` cookie and return it from the root loader
			- to set the cookie in the action
		- How does remix get to know that the cookie has been updated? (since cookie doesn't provide any pub/sub model or even access to JS if http only cookie). After the `post` request to call the action on the server, it also sends a get request automatically where we get the updated value of the cookie and the root component can depend on it.
	- ## Optimistic Theme
		- `useFetcher` with key option
			- Render the form from `fetcher.Form`
			- And then use the `useFetcher` in the `useTheme` hook to see ***what is being sent*** and then return from it. Otherwise return to the cookie value that the loader will return.
			- ```typescript
			  function useTheme() {
			  	const data = useLoaderData<typeof loader>()
			  	const themeFetcher = useFetcher({key: 'theme'})
			  	const optimisticTheme = themeFetcher?.formData?.get('theme')
			  	if (optimisticTheme === 'light' || optimisticTheme === 'dark') {
			  		return optimisticTheme
			  	}
			  	return data.theme
			  }
			  ```
	- ## Client Hints
	- Need:
		- preferences like reduced motion etc… are traditionally only known on the client. If someone has to animate something from opacity-0 to something, they can't do it for only server rendered app ( with js disabled ) – an empty thing would be shown without javascript.
	- Links
		- TODO [epicweb client hints](https://github.com/epicweb-dev/client-hints)
		- web.dev
			- TODO [Adapting to Users With Client Hints ](https://web.dev/articles/performance-optimizing-content-efficiency-client-hints)
			- TODO [User preference media features client hints headers](https://web.dev/articles/user-preference-media-features-headers)
			-
	-
- # Session Storage
  collapsed:: true
	- ## What do we want?
		- Cookie not accessible in JS `http-only` cookie
		- User not be able to change cookie. This is not possible. But we can detect when the cookie is not what we were expecting, by signing the cookie with crypto hash.
	- ## Cookie Attributes
		- ***path***: The path on the server for which the cookie is valid. Defaults to the current path. This means if the path is set to `/my-page`, the cookie will only be sent to the server when the user is on the `/my-page` path.
		- ***domain***: The domain for which the cookie is valid. Defaults to the current domain. This means if the domain is set to `example.com`, ==the cookie will be sent to the server **for all subdomains** of `example.com`==
		- ***expires***: The date and time when the cookie expires. If not set, the cookie will expire when the browser is closed.
		- ***max-age***: The number of seconds until the cookie expires. If not set, the cookie will expire when the browser is closed.
		- ***secure***: If set, the cookie ==will only be sent to the server over HTTPS.==
		- ***httpOnly***: If set, the cookie will not be accessible to JavaScript. ==This is useful for preventing cross-site scripting attacks.==
		- ***sameSite***: If set, the cookie will only be sent to the server if the request originated from the same site. ==This is useful for preventing cross-site request forgery attacks.==
			- *lax*: Cookie is sent
				- With 'safe' http methods ( GET )
				  logseq.order-list-type:: number
				- Only when request originates from the same site or when the user navigates to the URL from an external website.
				  logseq.order-list-type:: number
				- This is a balance between security and usability 
				  logseq.order-list-type:: number
					- preventing cookie from being sent with cross-site POST requests ( which are used for cross-site request forgery attacks )
					  logseq.order-list-type:: number
						- TODO [CSRF: OWASP Page](https://owasp.org/www-community/attacks/csrf)
					- Still allowing the user to access content via regular navigation
					  logseq.order-list-type:: number
	- ## In Remix
		- [Remix Docs: Sessions](https://remix.run/docs/en/main/utils/sessions)
			- Different storage adaptors for session storage.
			- Mostly only a good idea to send an id back to the client in a cookie.
			- ***Cookie flash pattern:*** temporary data stored in cookie.
	- ## Cookie flash
		- ### With the whole set unset shenanigans
			- in the note delete action
				- ```typescript
				  const toastCookieSession = await toastSessionStorage.getSession(
				  	request.headers.get('cookie'),
				  )
				  
				  toastCookieSession.set('toast', {
				  	type: 'success',
				  	title: 'Note deleted',
				  	description: 'Your note has been deleted',
				  })
				  
				  return redirect(`/users/${note.owner.username}/notes`, {
				  	headers: {
				  		'set-cookie': await toastSessionStorage.commitSession(toastCookieSession),
				  	},
				  })
				  ```
			- in the root
				- Get toast + unset the toast cookie session ( so that it doesn't show on refresh and is a one time thing )
					- ```typescript
					  const toastCookieSession = await toastSessionStorage.getSession(
					  	request.headers.get('cookie'),
					  )
					  const toast = toastCookieSession.get('toast')
					  toastCookieSession.unset('toast')
					  ```
				- setting in headers in second option in `json` utility
					- ```typescript
					  return json(
					  		{
					  			username: os.userInfo().username,
					  			theme: getTheme(request),
					  			toast,
					  			ENV: getEnv(),
					  			csrfToken,
					  			honeyProps,
					  		},
					  		{
					  			headers: combineHeaders(
					  				csrfCookieHeader ? { 'set-cookie': csrfCookieHeader } : null,
					  				{
					  					'set-cookie':
					  						await toastSessionStorage.commitSession(toastCookieSession),
					  				},
					  			),
					  		},
					  	)
					  ```
		- ### with `flash` common pattern already built in ( that encapsulates the above )
			- one wouldn't have to `unset` in the loader, if one `toastCookieSession.flash` instead of `set` in the action
			- ***An abstraction on top of this concept:***
				- ```typescript
				  // toast.server.ts
				  
				  import { createId as cuid } from '@paralleldrive/cuid2'
				  import { createCookieSessionStorage, redirect } from '@remix-run/node'
				  import { z } from 'zod'
				  import { combineHeaders } from './misc.tsx'
				  
				  export const toastKey = 'toast'
				  
				  const ToastSchema = z.object({
				  	description: z.string(),
				  	id: z.string().default(() => cuid()),
				  	title: z.string().optional(),
				  	type: z.enum(['message', 'success', 'error']).default('message'),
				  })
				  
				  export type Toast = z.infer<typeof ToastSchema>
				  export type ToastInput = z.input<typeof ToastSchema>
				  
				  export async function redirectWithToast(
				  	url: string,
				  	toast: ToastInput,
				  	init?: ResponseInit,
				  ) {
				  	return redirect(url, {
				  		...init,
				  		headers: combineHeaders(init?.headers, await createToastHeaders(toast)),
				  	})
				  }
				  
				  export async function createToastHeaders(toastInput: ToastInput) {
				  	const session = await toastSessionStorage.getSession()
				  	const toast = ToastSchema.parse(toastInput)
				  	session.flash(toastKey, toast)
				  	const cookie = await toastSessionStorage.commitSession(session)
				  	return new Headers({ 'set-cookie': cookie })
				  }
				  
				  export async function getToast(request: Request) {
				  	const session = await toastSessionStorage.getSession(
				  		request.headers.get('cookie'),
				  	)
				  	const result = ToastSchema.safeParse(session.get(toastKey))
				  	const toast = result.success ? result.data : null
				  	return {
				  		toast,
				  		headers: toast
				  			? new Headers({
				  					'set-cookie': await toastSessionStorage.destroySession(session),
				  				})
				  			: null,
				  	}
				  }
				  ```
		-
- # User Session
  collapsed:: true
	- ## Session Storage
		- ***in login action***
		  collapsed:: true
			- ```typescript
			  const submission = await parse(formData, {
			  		schema: intent =>
			  			LoginFormSchema.transform(async (data, ctx) => {
			  				if (intent !== 'submit') return { ...data, user: null }
			  
			  				const user = await prisma.user.findUnique({
			  					select: { id: true },
			  					where: { username: data.username },
			  				})
			  				if (!user) {
			  					ctx.addIssue({
			  						code: 'custom',
			  						message: 'Invalid username or password',
			  					})
			  					return z.NEVER
			  				}
			  				// verify the password (we'll do this later)
			  				return { ...data, user }
			  			}),
			  		async: true,
			  	})
			  	// get the password off the payload that's sent back
			  	delete submission.payload.password
			  
			  	if (submission.intent !== 'submit') {
			  		// @ts-expect-error - conform should probably have support for doing this
			  		delete submission.value?.password
			  		return json({ status: 'idle', submission } as const)
			  	}
			  	if (!submission.value?.user) {
			  		return json({ status: 'error', submission } as const, { status: 400 })
			  	}
			  
			  	const { user } = submission.value
			  
			  	const cookieSession = await sessionStorage.getSession(
			  		request.headers.get('cookie'),
			  	)
			  	cookieSession.set('userId', user.id)
			  
			  	return redirect('/', {
			  		headers: {
			  			'set-cookie': await sessionStorage.commitSession(cookieSession),
			  		},
			  	})})
			  ```
		- ***In root loader***
			- ```typescript
			  import { sessionStorage } from './utils/session.server.ts'
			  
			  const userId = cookieSession.get('userId')
			  const user = userId
			  	? await prisma.user.findUnique({
			  			select: {
			  				id: true,
			  				name: true,
			  				username: true,
			  				image: { select: { id: true } },
			  			},
			  			where: { id: userId },
			  		})
			  	: null
			  ```
- # Password Management
  collapsed:: true
	- ## Introduction
		- Don't encrypt password
		- Hash the password
			- should be slow
			- include a salt: random string added to password before hashing. Even if two users have same passwords, their salt will be different. Impossible for attacker to rainbow attack.
				- salt doesn't need to be stored anywhere.
		- What is hash? one way function. string -> given length string. Same input, same output, going from output to input is computationally difficult, impossible.
		- bcrypt: generates random salt.
	- ## Password Management
		- it's not possible to enforce a required value on both sides of a one-to-one relationship. So we can't ensure that a password is required on the `User` model at db level. But important tradeoff to accidentally leak passwords if they are a field on the `User` model itself instead of a separate model.
		- created password creating utility and used it in `seed.ts` to create passwords for test data users.
	- ## Sign Up
		- just added password field in user creation
- # Login
  collapsed:: true
	- timing attack, we won't be doing into this in this workshop. bcrypt compare function takes some time to execute.
	- going to use `useRouteLoaderData` to get loader data of parent.
	- ### Check if password is correct
	- ```typescript
	  // login action password check
	  
	  const userWithPassword = await prisma.user.findUnique({
	  	select: { id: true, password: { select: { hash: true } } },
	  	where: { username: data.username },
	  })
	  if (!userWithPassword || !userWithPassword.password) {
	  	ctx.addIssue({
	  		code: 'custom',
	  		message: 'Invalid username or password',
	  	})
	  	return z.NEVER
	  }
	  
	  const isValid = await bcrypt.compare(
	  	data.password,
	  	userWithPassword.password.hash,
	  )
	  
	  if (!isValid) {
	  	ctx.addIssue({
	  		code: 'custom',
	  		message: 'Invalid username or password',
	  	})
	  	return z.NEVER
	  }
	  
	  return { ...data, user: { id: userWithPassword.id } }
	  ```
	- ### Created the `useOptionalUser` and `useUser` using `useRouterLoaderData` and made `isOwner` `isMe` logic to lock UI.
		- ```typescript
		  import { useRouteLoaderData } from '@remix-run/react'
		  import { type loader as rootLoader } from '#app/root.tsx'
		  
		  export function useOptionalUser() {
		  	const data = useRouteLoaderData<typeof rootLoader>('root')
		  	return data?.user ?? null
		  }
		  
		  export function useUser() {
		  	const maybeUser = useOptionalUser()
		  	if (!maybeUser) {
		  		throw new Error(
		  			'No user found in root loader, but user is required by useUser. If user is optional, try useOptionalUser instead.',
		  		)
		  	}
		  	return maybeUser
		  }
		  ```
	-
- # Logout and Expiration
  collapsed:: true
	- ## Intro
		- logout button and automatic logout
			- `expires` (date at which cookie expires) and `max-age` number representing seconds the cookie should remain valid. Use anyone which feels okay, none is better than the other.
		- `session.unset('userId')` === logging out a cookie-managed session.
		- never logout on a get route. Always have a form that does a post request to `/logout`
		- **remember me** functionality === setting an expiration date which is long.
		- Another time to automatically logout user -> user's session is invalid. Primary reason -> user's account may have been deleted, in which case we need to send them to login page.
		- **Automatic logout:** Automatic logout is a little more complicated. If you wish to do this without client-side JavaScript, it involves setting a cookie with every single request and checking that cookie on subsequent requests. If the cookie is not present, then you can log the user out. It's a jarring experience and in the modern age, not likely necessary.
	- ## Logout
		- ```typescript
		  // logout.tsx route
		  
		  import { type ActionFunctionArgs, redirect } from '@remix-run/node'
		  import { validateCSRF } from '#app/utils/csrf.server.ts'
		  import { sessionStorage } from '#app/utils/session.server.ts'
		  
		  export async function loader() {
		  	return redirect('/')
		  }
		  
		  export async function action({ request }: ActionFunctionArgs) {
		  	const formData = await request.formData()
		  	await validateCSRF(formData, request.headers)
		  	const cookieSession = await sessionStorage.getSession(
		  		request.headers.get('cookie'),
		  	)
		  	return redirect('/', {
		  		headers: {
		  			'set-cookie': await sessionStorage.destroySession(cookieSession),
		  		},
		  	})
		  }
		  
		  ```
	- ## Expiration
		- ### Remember Me
			- in login and sign up
				- ```ts
				  'set-cookie': await sessionStorage.commitSession(cookieSession, {
				  	expires: remember ? getSessionExpirationDate() : undefined,
				  }),
				  ```
			- There is another way to make user experience even better but requires a little more work—to continue to update the expiration as the user is being on the website.
	- ## Deleted Users
		- ```ts
		  // 🐨 if there's a userId but no user then something's wrong.
		  // Let's delete destroy the session and redirect to the home page.
		  if (userId && !user) {
		  	// something weird happened... The user is authenticated but we can't find
		  	// them in the database. Maybe they were deleted? Let's log them out.
		  	throw redirect('/', {
		  		headers: {
		  			'set-cookie': await sessionStorage.destroySession(cookieSession),
		  		},
		  	})
		  }
		  ```
	- ## Automatic Logout
		- see the workshop notes, it's mostly using `useSubmit` to imperatively do a mutation/post request.
- # Protecting Routes
  collapsed:: true
	- ## Require Anonymous
		- ```typescript
		  export async function requireAnonymous(request: Request) {
		  	const userId = await getUserId(request)
		  	if (userId) {
		  		throw redirect('/')
		  	}
		  }
		  ```
		- Then use this for sign up and login action and loaders before anything else, so that logged in user doesn't visit those pages.
	- ## Require Authenticated
		- ```typescript
		  export async function requireUserId(request: Request) {
		  	const userId = await getUserId(request)
		  	if (!userId) {
		  		throw redirect('/login')
		  	}
		  	return userId
		  }
		  ```
		- Then use it where ever a logged in page is needed
	- ## Require Authorised
		- ```typescript
		  export async function requireUser(request: Request) {
		  	const userId = await requireUserId(request)
		  	const user = await prisma.user.findUnique({
		  		select: { id: true, username: true },
		  		where: { id: userId },
		  	})
		  	if (!user) {
		  		throw await logout({ request })
		  	}
		  	return user
		  }
		  ```
		- And use it with invariant to check user id with param id.
	- ## Redirect from Login
		- if a person is logged out, but in some tab they'd open some page to do some action, but as soon as they do they get shown login screen, they forget where they were and forget what they were trying to do. To prevent this `redirectTo` support.
		- ```ts
		  export async function requireUserId(request: Request, { redirectTo }: { redirectTo?: string | null } = {},) {
		  	const userId = await getUserId(request)
		  	if (!userId) {
		  		const requestUrl = new URL(request.url)
		  		redirectTo =
		  			redirectTo === null
		  				? null
		  				: redirectTo ?? `${requestUrl.pathname}${requestUrl.search}`
		  		const loginParams = redirectTo ? new URLSearchParams({ redirectTo }) : null
		  		const loginRedirect = ['/login', loginParams?.toString()]
		  			.filter(Boolean)
		  			.join('?')
		  		throw redirect(loginRedirect)
		  	}
		  	return userId
		  }
		  ```
		- Add a `redirectTo` input hidden field in login in / sign up form. and read it from there. Fill its `defaultValue` in the `useForm` options from:
			- ```ts
			  const [searchParams] = useSearchParams()
			  const redirectTo = searchParams.get('redirectTo')
			  ```
			- Reason Kent gave was not very exact, said relying on query params, sometimes don't get the value of query param in useFetcher or something, didn't really understand it fully.
		- Do a `return redirect(safeRedirect(redirectTo), {// headers})` in login and sign up action
- # Role-Based Access
  collapsed:: true
	- ## Roles Schema
		- ```prisma
		  model Permission {
		    id          String @id @default(cuid())
		    action      String // e.g. create, read, update, delete
		    entity      String // e.g. note, user, etc.
		    access      String // e.g. own or any
		    description String @default("")
		  
		    createdAt DateTime @default(now())
		    updatedAt DateTime @updatedAt
		  
		    roles Role[]
		  
		    @@unique([action, entity, access])
		  }
		  
		  model Role {
		    id          String @id @default(cuid())
		    name        String @unique
		    description String @default("")
		  
		    createdAt DateTime @default(now())
		    updatedAt DateTime @updatedAt
		  
		    users       User[]
		    permissions Permission[]
		  }
		  ```
		- Then prisma migrate dev
		- ## Roles Seed
		  collapsed:: true
			- This was tricky and confusing at first
			- These roles and permissions ( marked by commented values in prisma schema above) need to be in the database before we can use it to create any new user, or update existing users with the values.
			- This needs to happen for both dev and prod databases
			- Two ways to do this. One is to create sql by hand.
			- Two is to create a temp node script which creates all these permissions and roles in a temporary database
				- ```js
				  import { PrismaClient } from '@prisma/client'
				  import { execaCommand } from 'execa'
				  
				  const datasourceUrl = 'file:./tmp.ignored.db'
				  console.time('🗄️ Created database...')
				  await execaCommand('npx prisma migrate deploy', {
				  	stdio: 'inherit',
				  	env: { DATABASE_URL: datasourceUrl },
				  })
				  console.timeEnd('🗄️ Created database...')
				  
				  const prisma = new PrismaClient({ datasourceUrl })
				  
				  console.time('🔑 Created permissions...')
				  const entities = ['user', 'note']
				  const actions = ['create', 'read', 'update', 'delete']
				  const accesses = ['own', 'any']
				  for (const entity of entities) {
				  	for (const action of actions) {
				  		for (const access of accesses) {
				  			await prisma.permission.create({ data: { entity, action, access } })
				  		}
				  	}
				  }
				  console.timeEnd('🔑 Created permissions...')
				  
				  console.time('👑 Created roles...')
				  await prisma.role.create({
				  	data: {
				  		name: 'admin',
				  		permissions: {
				  			connect: await prisma.permission.findMany({
				  				select: { id: true },
				  				where: { access: 'any' },
				  			}),
				  		},
				  	},
				  })
				  await prisma.role.create({
				  	data: {
				  		name: 'user',
				  		permissions: {
				  			connect: await prisma.permission.findMany({
				  				select: { id: true },
				  				where: { access: 'own' },
				  			}),
				  		},
				  	},
				  })
				  console.timeEnd('👑 Created roles...')
				  
				  console.log('✅ all done')
				  
				  ```
			- Then we dump all the data from db to sql
				- `sqlite3 ./prisma/tmp.ignored.db .dump > tmp.ignored.sql`
				- make sure to remove all the unneeded initial prisma tables part. We just need the part where permissions and roles are being inserted, ignore all the rest. [local: Watch the video if it's not clear.](http://localhost:5639/exercise/08/02/solution?preview=diff). [Github repo](https://github.com/epicweb-dev/web-auth)
			- Then we copy everything from `temp.ignored.sql` and manually add it to the migration file for roles and permissionsThen we update seed file to assign user roles to generated users and 'admin' and 'user' role to kody
			- Then we reset database and run seed with it
			  id:: 67122b69-7324-4442-aacc-0808a8682598
				- ```sh
				  npx prisma migrate reset --force
				  ```
			- Make sure to create all new users with 'user' role from now on
				- ```json
				  roles: { connect: { name: 'user' } },
				  ```
				- ```json
				  roles: { connect: [{ name: 'admin' }, { name: 'user' }] },
				  ```
	- ## Delete Note
		- Check the [local tutorial diff](http://localhost:5639/exercise/08/03/solution?preview=diff), too much done
		- Summary:
			- instead of ownId === params.id invariant thrown check we build permission check
				- in the loader:
					- ```ts
					  const permission = userId
					  		? await prisma.permission.findFirst({
					  				select: { id: true },
					  				where: {
					  					roles: { some: { users: { some: { id: userId } } } },
					  					entity: 'note',
					  					action: 'delete',
					  					access: note.ownerId === userId ? 'own' : 'any',
					  				},
					  			})
					  		: null
					  ```
					- in the action
						- ```ts
						  	const permission = await prisma.permission.findFirst({
						  		select: { id: true },
						  		where: {
						  			roles: { some: { users: { some: { id: user.id } } } },
						  			entity: 'note',
						  			action: 'delete',
						  			access: note.ownerId === user.id ? 'own' : 'any',
						  		},
						  	})
						  ```
	- ## Permissions Utils
		- Permissions for each route in each loader and action could become very tedious. So we are going to build a utility which loads all permissions in the root. Make a `userHasRole` hook to access them.
		- ```ts
		  type Action = 'create' | 'read' | 'update' | 'delete'
		  type Entity = 'user' | 'note'
		  type Access = 'own' | 'any' | 'own,any' | 'any,own'
		  type PermissionString = `${Action}:${Entity}` | `${Action}:${Entity}:${Access}`
		  function parsePermissionString(permissionString: PermissionString) {
		  	const [action, entity, access] = permissionString.split(':') as [
		  		Action,
		  		Entity,
		  		Access | undefined,
		  	]
		  	return {
		  		action,
		  		entity,
		  		access: access ? (access.split(',') as Array<Access>) : undefined,
		  	}
		  }
		  
		  export function userHasPermission(
		  	user: Pick<ReturnType<typeof useUser>, 'roles'> | null,
		  	permission: PermissionString,
		  ) {
		  	if (!user) return false
		  	const { action, entity, access } = parsePermissionString(permission)
		  	return user.roles.some(role =>
		  		role.permissions.some(
		  			permission =>
		  				permission.entity === entity &&
		  				permission.action === action &&
		  				(!access || access.includes(permission.access)),
		  		),
		  	)
		  }
		  
		  export function userHasRole(
		  	user: Pick<ReturnType<typeof useUser>, 'roles'> | null,
		  	role: string,
		  ) {
		  	if (!user) return false
		  	return user.roles.some(r => r.name === role)
		  }
		  ```
		-
- # Managed Session
  collapsed:: true
	- ## Intro
	- Netflix like remote session management
	- ## Sessions Schema
		- ```ts
		  model Session {
		    id             String   @id @default(cuid())
		    expirationDate DateTime
		  
		    createdAt DateTime @default(now())
		    updatedAt DateTime @updatedAt
		  
		    user   User   @relation(fields: [userId], references: [id], onDelete: Cascade, onUpdate: Cascade)
		    userId String
		  
		    // non-unique foreign key
		    @@index([userId])
		  }
		  ```
	- ## Auth Utils
	  collapsed:: true
		- fetch user id from from session table instead of user table
			- ```ts
			  export async function getUserId(request: Request) {
			  	const cookieSession = await sessionStorage.getSession(
			  		request.headers.get('cookie'),
			  	)
			  	const sessionId = cookieSession.get(userIdKey)
			  	if (!sessionId) return null
			  	const session = await prisma.session.findUnique({
			  		select: { user: { select: { id: true } } },
			  		where: { id: sessionId, expirationDate: { gt: new Date() } },
			  	})
			  	if (!session?.user) {
			  		throw await logout({ request })
			  	}
			  	return session.user.id
			  }
			  ```
		- For the login auth utils there were a bunch of changes that only make sense once you see the whole code. I'm pasting the whole code here for peruse
			- ```ts
			  import { type Password, type User } from '@prisma/client'
			  import { redirect } from '@remix-run/node'
			  import bcrypt from 'bcryptjs'
			  import { safeRedirect } from 'remix-utils/safe-redirect'
			  import { prisma } from '#app/utils/db.server.ts'
			  import { combineResponseInits } from './misc.tsx'
			  import { sessionStorage } from './session.server.ts'
			  
			  const SESSION_EXPIRATION_TIME = 1000 * 60 * 60 * 24 * 30
			  export const getSessionExpirationDate = () =>
			  	new Date(Date.now() + SESSION_EXPIRATION_TIME)
			  
			  export const userIdKey = 'sessionId'
			  
			  export async function getUserId(request: Request) {
			  	const cookieSession = await sessionStorage.getSession(
			  		request.headers.get('cookie'),
			  	)
			  	const sessionId = cookieSession.get(userIdKey)
			  	if (!sessionId) return null
			  	const session = await prisma.session.findUnique({
			  		select: { user: { select: { id: true } } },
			  		where: { id: sessionId, expirationDate: { gt: new Date() } },
			  	})
			  	if (!session?.user) {
			  		// Perhaps user was deleted?
			  		cookieSession.unset(userIdKey)
			  		throw redirect('/', {
			  			headers: {
			  				'set-cookie': await sessionStorage.commitSession(cookieSession),
			  			},
			  		})
			  	}
			  	return session.user.id
			  }
			  
			  export async function requireUserId(
			  	request: Request,
			  	{ redirectTo }: { redirectTo?: string | null } = {},
			  ) {
			  	const userId = await getUserId(request)
			  	if (!userId) {
			  		const requestUrl = new URL(request.url)
			  		redirectTo =
			  			redirectTo === null
			  				? null
			  				: redirectTo ?? `${requestUrl.pathname}${requestUrl.search}`
			  		const loginParams = redirectTo ? new URLSearchParams({ redirectTo }) : null
			  		const loginRedirect = ['/login', loginParams?.toString()]
			  			.filter(Boolean)
			  			.join('?')
			  		throw redirect(loginRedirect)
			  	}
			  	return userId
			  }
			  
			  export async function requireAnonymous(request: Request) {
			  	const userId = await getUserId(request)
			  	if (userId) {
			  		throw redirect('/')
			  	}
			  }
			  
			  export async function requireUser(request: Request) {
			  	const userId = await requireUserId(request)
			  	const user = await prisma.user.findUnique({
			  		select: { id: true, username: true },
			  		where: { id: userId },
			  	})
			  	if (!user) {
			  		throw await logout({ request })
			  	}
			  	return user
			  }
			  
			  export async function login({
			  	username,
			  	password,
			  }: {
			  	username: User['username']
			  	password: string
			  }) {
			  	const user = await verifyUserPassword({ username }, password)
			  	if (!user) return null
			  	const session = await prisma.session.create({
			  		select: { id: true, expirationDate: true },
			  		data: {
			  			expirationDate: getSessionExpirationDate(),
			  			userId: user.id,
			  		},
			  	})
			  	return session
			  }
			  
			  export async function signup({
			  	email,
			  	username,
			  	password,
			  	name,
			  }: {
			  	email: User['email']
			  	username: User['username']
			  	name: User['name']
			  	password: string
			  }) {
			  	const hashedPassword = await getPasswordHash(password)
			  
			  	const session = await prisma.session.create({
			  		data: {
			  			expirationDate: getSessionExpirationDate(),
			  			user: {
			  				create: {
			  					email: email.toLowerCase(),
			  					username: username.toLowerCase(),
			  					name,
			  					roles: { connect: { name: 'user' } },
			  					password: {
			  						create: {
			  							hash: hashedPassword,
			  						},
			  					},
			  				},
			  			},
			  		},
			  		select: { id: true, expirationDate: true },
			  	})
			  
			  	return session
			  }
			  
			  export async function logout(
			  	{
			  		request,
			  		redirectTo = '/',
			  	}: {
			  		request: Request
			  		redirectTo?: string
			  	},
			  	responseInit?: ResponseInit,
			  ) {
			  	const cookieSession = await sessionStorage.getSession(
			  		request.headers.get('cookie'),
			  	)
			  	const sessionId = cookieSession.get(userIdKey)
			  	// delete the session if it exists, but don't wait for it, go ahead an log the user out
			      // the error could hit say for example when logout function is called if the user is already logged out and there is no session id.
			      // in that case we silently fail (since the user is already logged out)
			  	if (sessionId) {
			  		void prisma.session.deleteMany({ where: { id: sessionId } }).catch(() => {})
			  	}
			  	throw redirect(
			  		safeRedirect(redirectTo),
			  		combineResponseInits(responseInit, {
			  			headers: {
			  				'set-cookie': await sessionStorage.destroySession(cookieSession),
			  			},
			  		}),
			  	)
			  }
			  
			  export async function getPasswordHash(password: string) {
			  	const hash = await bcrypt.hash(password, 10)
			  	return hash
			  }
			  
			  export async function verifyUserPassword(
			  	where: Pick<User, 'username'> | Pick<User, 'id'>,
			  	password: Password['hash'],
			  ) {
			  	const userWithPassword = await prisma.user.findUnique({
			  		where,
			  		select: { id: true, password: { select: { hash: true } } },
			  	})
			  
			  	if (!userWithPassword || !userWithPassword.password) {
			  		return null
			  	}
			  
			  	const isValid = await bcrypt.compare(password, userWithPassword.password.hash)
			  
			  	if (!isValid) {
			  		return null
			  	}
			  
			  	return { id: userWithPassword.id }
			  }
			  ```
	- ## Session Cookie
		- Just changing the variable `userKey` to `sessionKey`
	- ## Delete Sessions
		- ```ts
		  async function signOutOfSessionsAction({ request, userId }: ProfileActionArgs) {
		  	const cookieSession = await sessionStorage.getSession(
		  		request.headers.get('cookie'),
		  	)
		  	const sessionId = cookieSession.get(sessionKey)
		  	invariantResponse(
		  		sessionId,
		  		'You must be authenticated to sign out of other sessions',
		  	)
		  	await prisma.session.deleteMany({
		  		where: {
		  			userId,
		  			id: { not: sessionId },
		  		},
		  	})
		  	return json({ status: 'success' } as const)
		  }
		  
		  // action
		  export async function action({ request }: ActionFunctionArgs) {
		  	const userId = await requireUserId(request)
		  	const formData = await request.formData()
		  	await validateCSRF(formData, request.headers)
		  	const intent = formData.get('intent')
		  	switch (intent) {
		  		case profileUpdateActionIntent: {
		  			return profileUpdateAction({ request, userId, formData })
		  		}
		          // if intent is sign out of all other sessions
		  		case signOutOfSessionsActionIntent: {
		  			return signOutOfSessionsAction({ request, userId, formData })
		  		}
		  		case deleteDataActionIntent: {
		  			return deleteDataAction({ request, userId, formData })
		  		}
		  		default: {
		  			throw new Response(`Invalid intent "${intent}"`, { status: 400 })
		  		}
		  	}
		  }
		  ```
- # Email
	-