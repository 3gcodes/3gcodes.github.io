+++
date = '2024-06-26T16:11:31-06:00'
draft = false
title = 'React Native AuthProvider'
summary = "There was a time when authentication was different in every single React Native application. Over time, the community / industry seems to have settled on a standard assuming no reliance on a third party library. Using a simple hook with context with some sort of secure storage mechanism for reloads is surprisngly simple and elegant."
[cover]
    image = 'cover.png'
keywords = ["Typescript", "React Native", "Authentication"]
+++


## Authentication

There was a time when authentication was different in every single React Native application. 
Over time, the community / industry seems to have settled on a standard assuming no reliance on a third party library. Using a simple hook with context with some sort of secure storage mechanism for reloads is surprisngly simple and elegant.

## The Provider

The following code is all you _need_ in your provider to get started. I've omitted any actual 
API calls to do the authentication. You may also want to provide additional functions for registration 
and logging out.

```Typescript
interface AuthProps {
    authState?: { token: string | null, authenticated: boolean | null };
    onLogin?: (email: string, password: string) => Promise<any>;
    onLogout?: () => Promise<any>;
}

const AuthContext = createContext<AuthProps>({});

export const useAuth = () => {
    return useContext(AuthContext);
}

export const AuthProvider = ( {children}: any ) => {
    const [authState, setAuthState] = useState<{
        token: string | null;
        authenticated: boolean | null;
    }>({ 
        token: null, 
        authenticated: null
    });

    const login = async (email: string, password: string) => {
      // do auth stuff and do put the token in the auth state
      setAuthState({
        token: tokenfromauth,
        authenticated: true
      });

      // do something here to store the token in a secure
      // peristed way so that it can be accessed if the app 
      // reloads. `SecureStorage` for example.
    }

    return <AuthContext.Provider value={onLogin: login, authState: authState}>{children}</AuthContext.Provider>
```

## App setup

In order for your application to display the login page when 
there is no token present or other pages when there is:

```Typescript
const App = () => {
  return (
    <AuthProvider>
       <MainContent />
    </AuthProvider>
  );
}

// this is necessary so that the provider is available
const MainContent = () => {
  const { authState } = useAuth();
  return (
    {authState?.authenticated ? <Text>Home</Text> : <LoginScreen />}
  );
}
```

## LoginScreen

The Login screen just needs to call the functional available in 
the AuthProvider

```Typescript
const { onLogin } = useAuth();

// call this from a button of some sort
const login = async () => {
  const result = await onLogin!(email, password);
}
```

## Conclusion

Obviously, there's a bit more nuance with regards to error handling and managing various other state information like whether or not the authentication API call is in progress or done, etc, but foundationally, this is all one needs.
