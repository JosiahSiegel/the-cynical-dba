---
layout: post
title: "What to do: All users are sysadmin"
permalink: what-to-do-all-users-are-sysadmin
date: 2017-07-09 19:04:24
comments: true
description: "A time bomb with a blast radius encompassing your career"
keywords: "MSSQLServer, permissions"
category: MSSQLServer

tags:
- permissions

---

Your first day on the job and all appears calm on the surface, until you learn that all **users were previously assigned a shared login with sysadmin permissions**. A time bomb with a blast radius encompassing your career.

## 1. Keep calm
   More than likely you were hired to help bring stability where you work. Do your best to appear in control...even though the opposite is true at the moment.
    
## 2. Develop the solution
   
   Even with limited understanding of the business's structure, you can create a primitive permissions solution.
   
   List all positions that you know directly access the database (developer, analyst, manager, etc). **Congratulations, these are your new database roles!**
     
   Imagine which permissions will be needed within each role. Management can help decide specifics later.   
 
## 3. Reveal to management
   Give them the facts. Let them know that a system catastrophe is on the horizon, but that you have the solution.
   
   Reveal your new database roles and ask if they encompass all positions that require access.
   If so, request the names (or how to acquire the names) of all users that will fall within these roles, as well as the permissions these positions require (permissions will likely be inaccurate, but it's a work in progress).

## 4. Build the solution
   With the roles, estimated permissions, and assigned names in hand, get started by first adding a login for each user (Windows Authentication if available) to the instance. If you are creating SQL logins, be sure to check "*user must change password at next login*" to help ensure that only the user of the login knows the password.
    
## 5. Migrate users
   Request management to confirm users have database access with their new logins before removing the community super login. There will be the inevitable "*I couldn't login once, so this will never work*" pushback, as well as wrong password errors. Using Windows Authentication is great in this regard, as the users will undoubtedly know their password.
    
> There are additional security concerns to cover (password rotation/complexity, which permissions to assign, etc), but these steps will get you started in the right direction.

