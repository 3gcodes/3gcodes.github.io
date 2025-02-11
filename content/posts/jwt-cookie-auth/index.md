+++
date = '2025-02-10T00:01:01-06:00'
draft = false
title = 'JWT Cookie Auth with Spring Security'
summary = ""
categories = ['tech']
tags = ['spring', 'spring-security', 'spring-boot', 'auth', 'jwt', 'cookies']
keywords = ["spring", "spring-boot", "spring-security", "auth", "jwt", "cookies"]
[cover]
    image = 'cover.png'
+++

### Intro

I've been working on a Board Game Collection side project. The backend is Spring Boot with a Neo4J database. Mostly, I use this project as a learning tool as well as for personal use. I built a small mobile app as the client in Flutter but over time realized native mobile was unnecessary. So as another learning opportunity I've been bouncing back and forth between using JTE with HTMX as well as learning Svelte.

My first step was to get a login page working and I immediately ran into a snag. As it happened, it's a very similar issue for both HTMX and Svelte; JWT. For HTMX, since its use-case is not really an SPA, I wasn't sure how to handle the page requests if I just had a JWT. I didn't want to store it in LocalStorage as it's not that secure, even though I am signing my JWT with RSA. Similarly, with Svelte (specifically Sveltekit), there's not a straightforward way to wire in the Authorization header into fetch. Svelte has hooks; client, server, shared. But the `handleFetch` hook, which would have been exactly what I needed, only works for server and not client.

The solution? Cookies. However, I also wanted to keep my API stateless. For the remainder of this post I'm going to assume a lot. I don't want to bog this down with a ton of code from start to finish. The assumption I'm going to make is that you already know Spring Boot and Spring Security and that you're already managing JWT's in some way and maybe you're just curious about how to use cookies with JWT.

### Creating the Cookie

If we're going to use cookies, we need to create one. Inside my `AuthController` I've modified the `login` method to create the cookie on the `HttpServletResponse`.

```java
@PostMapping("/login")
public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest payload, HttpServletResponse response) {
	final var authResponse = authService.login(payload.username(), payload.password());
	if (authResponse.token() == null) {
		return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(authResponse);
	} else {
		Cookie jwtCookie = new Cookie("jwt", authResponse.token());
		jwtCookie.setHttpOnly(true);
		jwtCookie.setSecure(true);
		jwtCookie.setPath("/");
		jwtCookie.setMaxAge(3600);
		response.addCookie(jwtCookie);
		return ResponseEntity.ok(authService.login(payload.username(), payload.password()));
	}
}
```
Now that we have our cookie, we need to use it to get our JWT and authenticate into Spring Security.

### Using the Cookie

```java
@Component
public class JwtCookieFilter extends OncePerRequestFilter {

	private final Logger LOG = LoggerFactory.getLogger(JwtCookieFilter.class);

	private final RsaKeyProperties rsaKeyProperties;

	public JwtCookieFilter(RsaKeyProperties rsaKeyProperties) {
		this.rsaKeyProperties = rsaKeyProperties;
	}

	@Override
	protected void doFilterInternal( //
			HttpServletRequest request, //
			HttpServletResponse response, //
			FilterChain filterChain) throws ServletException, IOException {
		Cookie[] cookies = request.getCookies();
		if (cookies != null) {
			for (Cookie cookie : cookies) {
				if ("jwt".equals(cookie.getName())) {
					String token = cookie.getValue();

					try {
						SignedJWT signedJWT = SignedJWT.parse(token);
						if (isValidToken(signedJWT)) {
							JWTClaimsSet claims = signedJWT.getJWTClaimsSet();
							final var username = claims.getSubject();
							final var userDetails = new AuthUser();
							userDetails.setUsername(username);
							userDetails.setRoles(Set.of(new AuthRole().withAuthority("ROLE_USER")));
							UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());

							SecurityContextHolder.getContext().setAuthentication(authentication);
						}
					} catch (ParseException | JOSEException e) {
						// don't really care about this, 401 will fire which is what we want
						LOG.error(e.getMessage());
					}
				}
			}
		}
		filterChain.doFilter(request, response);
	}

	private boolean isValidToken(SignedJWT signedJWT) throws ParseException, JOSEException {
		JWSVerifier verifier = new RSASSAVerifier(rsaKeyProperties.publicKey());
		final var verified = signedJWT.verify(verifier);
		final var expired = !isTokenExpired(signedJWT);
		return verified && !expired;
	}

	private boolean isTokenExpired(SignedJWT signedJWT) throws ParseException {
		final var expiration = signedJWT.getJWTClaimsSet().getExpirationTime();
		return expiration != null && expiration.after(new Date());
	}

	@Override
	protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
		String path = request.getServletPath();
		return path.startsWith("/auth/") || path.startsWith("/error");
	}
}
```

There are a few key points to call out here.

1. In the `doFilterInternal` method, I find the cookie with the name `jwt`. If I don't find it, the filter just continues, and Spring Security does the right thing and throws a 401. This is a horrible name. Please make it something more unique in your own code.
2. If I do find the cookie, I verify that is is valid.
3. Once the cookie is validated, if it is valid, I force a `UserDetail` object into the `SecurityContextHolder`. I'm hard coding authorities right now, but will be replacing that with role data in the JWT eventually.
4. If the JWT isn't valid, again, Spring Security does the right thing and returns a 401. 
5. The `shouldNotFilter` method is important because this filter needs to permit non-authenticated requests for specific routes. I don't love this, but I don't see another way around it. I might move some of this wiring into property config at some point.

### Wire up the Filter into the Security Config

Adjust your Security Config class. You'll need a bean for the `JwtCookieFilter`:

```java
@Bean
JwtCookieFilter jwtCookieFilter() {
    return new JwtCookieFilter(rsaKeyProperties);
}
```

My `RsaKeyProperties` looks like this:

```java
@ConfigurationProperties(prefix = "rsa")
public record RsaKeyProperties(RSAPublicKey publicKey, RSAPrivateKey privateKey) {
}
```

In your Security Config class, where you have a `securityFilterChain` method, add the following to the `HttpSecurity` object.

```java
.addFilterBefore(jwtCookieFilter(), UsernamePasswordAuthenticationFilter.class)
```

I should also note that you're going to need CORS configured correctly.

### The Svelte Login

In my client code, I have the following fetch call to authenticate.

```typescript
const response = await fetch('http://localhost:8080/auth/login', {
	method: 'POST',
	headers: {'Content-Type': 'application/json' },
	body: JSON.stringify({username: form.username, password: form.password})
});
```

On success, the cookie is created and the browser does the right thing to manage it. On subsequent fetch calls to protected routes, you just want to make sure that `credentials: 'include' is set. So to get my list of games I'd do the following:

```typescript
const response = await fetch('http://localhost:8080/api/games', {
	method: 'GET',
	headers: {'Content-Type': 'application/json'},
	credentials: 'include'
});
```

HTMX has `htmx.config.withCredentials` that is used to wire it up correctly.

### Finally

Again, I assumed that you already had some level of JWT working with Spring Security. There are plenty of articles and videos on this topic. Google is your friend. 

My understanding is that this is secure enough though I'm not sure I'd do it this way for a commercial app. And honestly, I'd probably not even roll my own authentication. It's a huge liability.

I do hope this helps as I spent a lot of time trying to track down all the right code to make this work.



