- One common example of this is error messages. The web doesn't have a built-in mechanism for associating error messages to the fields they're reporting on, so you need to use ARIA to do this.
  
  ```html
  <form>
  	<label for="name">Name</label>
  	<input
  		id="name"
  		type="text"
  		aria-invalid="true"
  		aria-describedby="name-error"
  	/>
  	<div id="name-error">Name is required</div>
  </form>
  ```
- Associate a button outside of a form with the form:
  id:: 66ec1bb7-b0ea-49d9-b8e2-aa08e21faefd
  
  ```html
  <button form="form-id" type="reset">
  	Reset
  </button>
  ```
- `noValidate={isHydrated}`: browser checks by default with its validation attributes, and when the app hydrates, we use javascript to take over the responsibility so that we can show our own custom error UI.
- see [this file](https://github.com/epicweb-dev/web-forms/blob/main/exercises/02.accessibility/03.solution.focus/app/routes/users%2B/%24username_%2B/notes.%24noteId_.edit.tsx) to revise how to do focus management for form and it's fields which error out. A hint to jog memory
  
  ```typescript
  	useEffect(() => {
  		const formEl = formRef.current
  		if (!formEl) return
  		if (actionData?.status !== 'error') return
  
  		if (formEl.matches('[aria-invalid="true"]')) {
  			formEl.focus()
  		} else {
  			const firstInvalid = formEl.querySelector('[aria-invalid="true"]')
  			if (firstInvalid instanceof HTMLElement) {
  				firstInvalid.focus()
  			}
  		}
  	}, [actionData])
  ```
-
- …(there were a lot of things here about conform and handling complex fields and stuff, refer to the course itself, was too much information to type)
- ## Honeypot
	- Show an empty field in the form, bunch of things can be done on it
		- If it's sent by the user, it's most likely sent by a bot
		- > Another useful thing you can do along with the honeypot field is to add a field that will allow you to determine when the form was generated. So if the form is submitted too quickly, you can be pretty confident that it's a bot. This is more tricky than it sounds because bots could easily change the value of that field, so you do need to encrypt the value. But if you manage that, it's a great way to catch bots.
		- `remix-utils` has honeypot utils
- ## CSRF
	- > That's a CSRF attack in action. The attack exploits the trust a site has in the user's browser and can have damaging effects.
	- ### Foundations
		- Server: Ye lo token client sahab
		  Client: okay, mai bhejta jab user kuch submit karta
		  Attacker: hehehehe, click this button, hehehehe
		  Server: some data has come in a POST api call, but where's the token? Ehhh! Rejected!
		- `SameSite: Strict | Lax` === cookie isn't sent in cross site requests.
		- Check `Origin` and `Referer`
		- Encourage users to logout, especially in public or shared computers.