# Azure DevOps

## Notes

- You can organize your pipeline into jobs. Every pipeline has at least one job. A job is a series of steps that run sequentially as a unit. In other words, a job is the smallest unit of work that can be scheduled to run. [ref][8]

- For self-hosted parallel jobs, you'll start by deploying our self-hosted agents on your machines. You can register any number of these self-hosted agents in your organization. [ref][8]

- You cannot run Mac agents using **scale sets**. You can only run Windows or Linux agents this way. You should not associate a VMSS to multiple pools. [ref][5]

## Additional Information

- Price
- [Azure | MS | Pricing for Azure DevOps][7]
- [Docs | MS | Configure and pay for parallel jobs][1]
- Jobs
- [Docs | MS | Specify jobs in your pipeline][8]
- [Docs | MS | Deployment jobs][9]
- [Docs | MS | Define container jobs (YAML)][10]
- Self Hosted Agents
- [Docs | MS | Self-hosted Linux agents][2]
- [Docs | MS | Self-hosted macOS agents][3]
- [Docs | MS | Self-hosted Windows agents][4]
- [Docs | MS | Azure virtual machine scale set agents][5]
- [Docs | MS | Run a self-hosted agent in Docker][6]

[1]: https://docs.microsoft.com/en-us/azure/devops/pipelines/licensing/concurrent-jobs
[2]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux
[3]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-osx
[4]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows
[5]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/scale-set-agents
[6]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker
[7]: https://azure.microsoft.com/en-us/pricing/details/devops/azure-devops-services/
[8]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/phases
[9]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/deployment-jobs
[10]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/container-phases
