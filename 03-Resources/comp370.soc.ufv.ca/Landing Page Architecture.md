                         INTERNET
                            │
                            │ HTTPS
                            ▼
                  comp370.soc.ufv.ca
                   198.162.116.20
                            │
                            │ UFV NAT / Gateway
                            ▼
                     Your VM
                  172.30.255.29
                            │
                            ▼
                    ┌──────────────┐
                    │    NGINX     │
                    │ Reverse Proxy│
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
          Landing       Project A    Project B
           :3000          :3001        :3002


## Public URLs

Your landing page would be:

https://comp370.soc.ufv.ca/

Project links:

https://comp370.soc.ufv.ca/projects/project-a/

https://comp370.soc.ufv.ca/projects/project-b/

https://comp370.soc.ufv.ca/projects/project-c/

Users never need:

:3001

:3002

:3003

---

# The landing page doesn't need to know about ports

This is actually one of the biggest benefits.

Your Next.js/React landing page could simply have:

const projects = [

  {

    id: "project-a",

    name: "AI Code Analyzer",

    url: "/projects/project-a/"

  },

  {

    id: "project-b",

    name: "Testing Dashboard",

    url: "/projects/project-b/"

  }

]

The browser sees:

/project-a/

NGINX handles the mapping:

/project-a/

      ↓

project-a container :3000

So the architecture becomes:

                 LANDING PAGE

                      │

                      │

            /projects/project-a/

                      │

                      ▼

                    NGINX

                      │

                      ▼

               project-a:3000

---

# I would actually improve the previous architecture

Instead of running:

Landing     :3000

Project A   :3001

Project B   :3002

Project C   :3003

I recommend Docker networking where **every application uses port 3000 internally**:

                  NGINX

                    │

        ┌───────────┼───────────┐

        │           │           │

        ▼           ▼           ▼

     landing     project-a   project-b

       :3000        :3000       :3000

Docker distinguishes them by service name.

For example:

http://landing:3000

http://project-a:3000

http://project-b:3000

This is much cleaner.

---

# Your landing page becomes a service registry

This is the architecture I'd ultimately aim for:

                     COMP 370 PORTAL

                            │

                            ▼

                     comp370.soc.ufv.ca

                            │

                            ▼

                          NGINX

                            │

        ┌───────────────────┼───────────────────┐

        │                   │                   │

        ▼                   ▼                   ▼

       "/"          "/projects/ai/"     "/projects/test/"

        │                   │                   │

        ▼                   ▼                   ▼

    Landing             AI Project          Test Project

     :3000                 :3000               :3000

Your landing page is therefore **not responsible for networking**.

It is simply the catalog/discovery layer.

NGINX is responsible for routing.

Docker is responsible for service isolation/networking.