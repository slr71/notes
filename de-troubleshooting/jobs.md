# Job Troubleshooting

## Query to Get Exit Statuses

``` sql
select parent.job_name, child.job_name, jsu.message
from jobs parent
join users u on parent.user_id = u.id
join jobs child on parent.id = child.parent_id
join job_steps s on child.id = s.job_id
join job_status_updates jsu on s.external_id = jsu.external_id
and u.username = 'kmichel@iplantcollaborative.org'
and parent.status = 'Running'
and parent.job_name like '4-16-18%'
and child.status = 'Failed'
and jsu.message like '%exit status%'
order by parent.job_name, child.job_name;
```
