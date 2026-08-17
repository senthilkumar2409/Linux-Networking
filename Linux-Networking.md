# Linux Networking:
<img alt="OSI-Model" height="745" loading="lazy" src="https://media.geeksforgeeks.org/wp-content/uploads/20241111182857579134/OSI-Model.gif" width="1042" title="Click to enlarge" style="cursor: zoom-in;">
### OSI Model:

**Application Layer:**

  - The layer closest to the end user. It provides network services directly to applications — web browsing, email, file transfer.

  - Protocols: HTTP/HTTPS, FTP, SMTP, DNS
  - Example: When you type a URL, this is where the browser's request begins.

**Presentation Layer:**

  - Presentation Layer ensures that data is formatted, secured, and compressed so that the receiving application can correctly interpret it.
  - Data's are encrypted in this layer, while receiving end decryption happens here.

**Session Layer:**

  - A session is created in this layer, between browser and client server, Once the request is authenticated with client server session is created.
  - establishment, maintenance, and termination of connections between applications.
  - These Application, Presentation and session layer happens at browser level

**Transport Layer:**

  - Data is broken down into small segments or packets.
  - decides the wheather request is tcp or udp protocol
  - tcp needs handshake to establish a connection and udp doesn't requires.

**Network Layer:**

  - It happens on our home router
  - It has a sender and receiver ip address to find a shortest routing path to reach the receiver server.

**Data Link Layer:**

 - At this layer, data packets received from the Network layer are divided or converted into manageable units called Frames. This is done by adding specific "start" and "stop" bits so the receiver can recognize where one unit of data ends and the next begins.

 - Addressing and Hardware: The DLL uses MAC addresses (Physical Addressing) to identify hosts. It encapsulates the sender’s and receiver’s MAC addresses into the frame header. Common devices operating here are Switches and Bridges.

**Physical Layer:**

 - Once data reaches routers/switches it transfer to optical fibre cable(It understands electronical signal)
 - So, Data's are transfer as electronical signals to devices.
