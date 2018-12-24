---
layout: post
title:  "Self-documenting Service interfaces"
date:   2011-11-23 10:11:45 +0100
categories: java
---

I’m mocking up a quick example project for a presentation about Fitnesse and when I had to draw up some quick business services I was once again reminded of how much I hate Exceptions.

Let me explain: Exceptions are a pain to create and to handle… often you end up throwing some kind of generic BusinessException and stuffing a String in there or at the most making a subclass for the business method. Making a different Exception for each thing that can go wrong is not only tiresome to write, it also makes using the interface a pain because of the try {} catch structure needed.

So… here’s what I came up with :)

We start out really simple, with a Result enum:

```java
public enum Result {
 
    /** The operation succeeded **/
    SUCCESS,
 
    /**
    * The operation failed because a rule was not followed.
    * E.g. the user didn't fill in a field, the name exists,
    * etc.
    */
    FAILURE,
 
   /**
    * The operation failed for an unexpected reason.
    * It's usually better to throw an exception in that
    * case.
    */
    ERROR
}
```
Nothing special so far. The second class mostly functions as a Generic wrapper for the Result:

```java
   /**
   * Result of an operation
   **/
    public class OperationResult<P, F> {
        /** Value if succesful **/
        private P value;
 
        /** Reason for failure **/
        private F reason;
 
        /** Result: SUCCESS, FAIL, ERROR **/
        private Result result;
 
        public OperationResult( Result result ) {
            this.result = result;
        }
 
        public OperationResult( Result result, P value, F reason ) {
            this(result);
            this.value = value;
            this.reason = reason;
        }
 
        public void setReason(F reason) { 
            this.reason = reason;
        }
 
        public void setResult(Result result) {
            this.result = result;
        }
 
        public void setValue( P value ) {
            this.value = value;
        }
 
        public F getReasonForFailure() {
            return reason;
        }
 
        public P getValue() {
            return value;
        }
 
        public Result getResult() {
            return result;
        }
 
        public boolean isSuccess() {
            return this.result == Result.SUCCESS;
        }
 
        public boolean isError() {
            return this.result == Result.ERROR;
        }
 
        public boolean isFailure() {
            return this.result == Result.FAILURE;
        }
}
```
OK, now comes the “clever” bit:

This allows me to write my business interface as follows:

```java
 public interface UserService {
 
   /**
    * All the failures that can occur while creating a new User
   */
    public static enum UserCreationFailure {
        EMAIL_NOT_UNIQUE, /** The username already exists in the system **/ 
       EMAIL_INVALID, /** The provided e-mail was not a valid mail address **/
       INVALID_PASSWORD; /** The password should was not valid **/
    }
 
/**
* Adds a new user to the system.
*
* The new user is returned. If the user could not
* be created, the reason is returned.
*
* @param emailAddress
* @param realName
* @param password
*
* @return on success the created user, or the reason of the failure.
*/
OperationResult<User, UserCreationFailure>
createNewUser(String emailAddress,
String realName, String password );
 
}
```

The interface documents exactly what can go wrong. If the operation succeeds you get the value, else you get the reason for failure:

```java
OperationResult<User, UserService.UserCreationFailure> result = 
  this.userService.createNewUser( emailAddress, name, password);
 
User newUser = null;
 
if ( result.isSuccess() ) {
    newUser = result.getValue();
} else {
    UserService.UserCreationFailure reasonForFailure =   
        result.getReasonForFailure();
        //Do something with the reason
}
```
