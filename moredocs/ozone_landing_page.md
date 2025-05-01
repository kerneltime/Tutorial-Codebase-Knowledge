# Apache Ozone: Scalable, Redundant, and Cloud-Native Object Storage

Apache Ozone is a distributed object store designed for Hadoop and cloud-native environments. It excels at managing billions of objects of varying sizes, making it ideal for modern data-intensive applications.

## Key Features and Benefits

*   **Multi-Protocol Support:** Ozone supports both S3 and Hadoop File System APIs, providing flexibility for developers and seamless integration with existing tools and workflows.
*   **Scalability:** Designed to scale to tens of billions of files and blocks, Ozone can handle the demands of even the largest datasets.
*   **Consistency:** Ozone provides strong consistency guarantees, ensuring data integrity and reliability.
*   **Cloud-Native Architecture:** Ozone is designed to thrive in containerized environments like Kubernetes and YARN, simplifying deployment and management.
*   **Security:** Ozone integrates with Kerberos for authentication, supports native ACLs, and integrates with Ranger for access control, ensuring data is protected.
*   **High Availability:** Ozone's fully replicated architecture is designed to withstand multiple failures, ensuring continuous data availability.

## Why Choose Ozone?

Ozone offers a compelling solution for developers and storage admins seeking a scalable, reliable, and secure object store. Whether you're building cloud-native applications, managing big data workloads, or simply need a robust storage solution, Ozone has you covered.

## Get Started

Ready to explore Ozone? Here's how to get started:

*   **Run Ozone from a Docker Image:** The easiest way to get started is with our pre-built Docker image:

    ```
    docker run -p 9878:9878 apache/ozone
    ```

*   **Download a Release:** For a more realistic cluster setup, download the latest release and use Docker Compose:

    ```
    cd compose/ozone
    docker-compose up -d --scale datanode=3
    ```

*   **Kubernetes Deployment:** Ozone is a first-class citizen in cloud-native environments. Use our Kubernetes resource files for easy deployment.

## Learn More

*   **Documentation:** [https://ozone.apache.org/docs/](https://ozone.apache.org/docs/)
*   **Website:** [https://ozone.apache.org](https://ozone.apache.org)
*   **Community:** Join the Ozone community on Slack ([http://s.apache.org/slack-invite](http://s.apache.org/slack-invite)) or GitHub Discussions ([https://github.com/apache/ozone/discussions](https://github.com/apache/ozone/discussions)).

## Contribute

We welcome contributions! Open a Jira issue and create a pull request to get involved.

## License

Apache Ozone is licensed under the Apache 2.0 License.