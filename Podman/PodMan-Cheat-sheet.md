## Podman Cheat Sheet

### 1. Most Used Podman Commands
The most commonly used Podman commands, grouped by type:

#### **Container Commands**
| **Command**       | **Description**                                   | **Example**                                   |
|--------------------|---------------------------------------------------|-----------------------------------------------|
| `podman run`      | Run a container from an image.                    | `podman run -it IMAGE_NAME`                   |
| `podman stop`     | Stop a running container.                         | `podman stop CONTAINER_ID`                    |
| `podman rm`       | Remove a container.                               | `podman rm CONTAINER_ID`                      |
| `podman exec`     | Execute a command in a running container.         | `podman exec -it CONTAINER_ID COMMAND`        |
| `podman logs`     | View logs of a container.                         | `podman logs CONTAINER_ID`                    |
| `podman inspect`  | Inspect details of a container or image.          | `podman inspect CONTAINER_ID_OR_IMAGE_NAME`   |
| `podman port`     | List port mappings for a container.               | `podman port CONTAINER_ID`                    |

#### **Image Commands**
| **Command**       | **Description**                                   | **Example**                                   |
|--------------------|---------------------------------------------------|-----------------------------------------------|
| `podman images`   | List available images.                            | `podman images`                               |
| `podman pull`     | Pull an image from a registry.                    | `podman pull IMAGE_NAME`                      |
| `podman rmi`      | Remove an image.                                  | `podman rmi IMAGE_NAME`                       |

#### **Network Commands**
| **Command**       | **Description**                                   | **Example**                                   |
|--------------------|---------------------------------------------------|-----------------------------------------------|
| `podman ps`       | List running containers.                          | `podman ps`                                   |

---

### 2. Advanced Podman Commands

#### **Advanced Image Commands**
| **Command**       | **Description**                                   | **Example**                                   |
|--------------------|---------------------------------------------------|-----------------------------------------------|
| `podman build`    | Build an image from a Dockerfile.                 | `podman build -t IMAGE_NAME .`                |
| `podman tag`      | Tag an image with a new name.                     | `podman tag SOURCE_IMAGE NEW_IMAGE`           |
| `podman push`     | Push an image to a registry.                      | `podman push IMAGE_NAME`                      |
| `podman save`     | Save an image to a tar archive.                   | `podman save -o FILE_NAME IMAGE_NAME`         |
| `podman load`     | Load an image from a tar archive.                 | `podman load -i FILE_NAME`                    |

#### **Advanced Network Commands**
| **Command**               | **Description**                                   | **Example**                                   |
|----------------------------|---------------------------------------------------|-----------------------------------------------|
| `podman network create`    | Create a new network.                             | `podman network create NETWORK_NAME`          |
| `podman network rm`        | Remove an existing network.                       | `podman network rm NETWORK_NAME`              |
| `podman network inspect`   | Inspect details of a network.                    | `podman network inspect NETWORK_NAME`         |
| `podman network ls`        | List all available networks.                      | `podman network ls`                           |
| `podman network connect`   | Connect a container to a network.                | `podman network connect NETWORK_NAME CONTAINER_ID` |
| `podman network disconnect`| Disconnect a container from a network.           | `podman network disconnect NETWORK_NAME CONTAINER_ID` |

---

### 3. Podman Persistent Storage: Concepts and Best Practices

#### **Key Concepts**
- **Volumes**: Podman-managed storage locations that persist data independently of container lifecycles.
- **Bind Mounts**: Map a host directory or file to a container for direct access.
- **OverlayFS**: Efficient filesystem layering used by Podman.

#### **Best Practices**
1. Use named volumes for portability: `podman volume create my_volume`.
2. Avoid storing critical data in containers; use volumes or bind mounts.
3. Secure sensitive data with proper permissions.
4. Regularly back up persistent data: `podman volume export my_volume > backup.tar`.
5. Use bind mounts for development: `podman run -v /host/path:/container/path IMAGE_NAME`.
6. Monitor volume usage: `podman volume inspect my_volume`.

#### **Common Commands**
| **Command**               | **Description**                                   | **Example**                                   |
|----------------------------|---------------------------------------------------|-----------------------------------------------|
| `podman volume create`     | Create a new named volume.                        | `podman volume create my_volume`              |
| `podman volume ls`         | List all available volumes.                       | `podman volume ls`                            |
| `podman volume inspect`    | Inspect details of a volume.                      | `podman volume inspect my_volume`             |
| `podman volume rm`         | Remove a named volume.                            | `podman volume rm my_volume`                  |
| `podman volume export`     | Export a volume to a tar archive.                 | `podman volume export my_volume > backup.tar` |
| `podman volume import`     | Import a volume from a tar archive.               | `podman volume import my_volume < backup.tar` |

---

### 4. Podman Port Mapping: Concepts and Best Practices

#### **Key Concepts**
- **Host Port**: Exposed port on the host machine.
- **Container Port**: Port inside the container where the application listens.
- **Port Mapping**: Links a host port to a container port using `-p` or `--publish`.

#### **Best Practices**
1. Use explicit port mappings: `podman run -p 8080:80 IMAGE_NAME`.
2. Avoid privileged ports (below 1024); use higher numbers like `8080`.
3. Restrict access to specific interfaces: `podman run -p 127.0.0.1:8080:80 IMAGE_NAME`.
4. Document port usage to avoid conflicts.
5. Use firewalls to secure exposed ports.
6. Test port accessibility with tools like `curl` or `telnet`.

#### **Common Commands**
| **Command**       | **Description**                                   | **Example**                                   |
|--------------------|---------------------------------------------------|-----------------------------------------------|
| `podman run -p`   | Map a host port to a container port.               | `podman run -p 8080:80 IMAGE_NAME`            |
| `podman port`     | List port mappings for a running container.        | `podman port CONTAINER_ID`                    |

---

### 5. Creating a Container Image from a Dockerfile

#### **Steps to Build an Image**
1. **Create a Dockerfile**:
    - Example:
      ```dockerfile
      FROM alpine:latest
      RUN apk add --no-cache curl
      CMD ["curl", "--version"]
      ```

2. **Build the Image**:
    - Command: `podman build -t my_custom_image .`

3. **Verify the Image**:
    - Command: `podman images`

#### **Best Practices**
- Use descriptive tags: `podman build -t my_app:1.0 .`.
- Minimize image size by using lightweight base images like `alpine`.
- Test the image: `podman run -it my_custom_image`.
- Clean up intermediate images: `podman build --rm -t my_custom_image .`.

#### **Common Commands**
| **Command**       | **Description**                                   | **Example**                                   |
|--------------------|---------------------------------------------------|-----------------------------------------------|
| `podman build`    | Build an image from a Dockerfile.                 | `podman build -t IMAGE_NAME .`                |
| `podman tag`      | Tag an image with a new name.                     | `podman tag SOURCE_IMAGE NEW_IMAGE`           |
| `podman images`   | List available images.                            | `podman images`                               |
| `podman rmi`      | Remove an image.                                  | `podman rmi IMAGE_NAME`                       |

