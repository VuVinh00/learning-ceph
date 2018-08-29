# Config multi-stite for Rados-gateway

## tren cluster 1
radosgw-admin realm create --rgw-realm=cookbookv2 --default

radosgw-admin zonegroup create --rgw-zonegroup=us --endpoints=http://us-east-1.ceph-s3.com:80 \
--rgw-realm=cookbookv2 --master --default

radosgw-admin zone create --rgw-zonegroup=us \
--rgw-zone=us-east-1 --master --default \
--endpoints=http://us-east-1.ceph-s3.com:80

radosgw-admin zonegroup remove --rgw-zonegroup=default --rgw-zone=default
radosgw-admin zone delete --rgw-zone=default

radosgw-admin period update --commit

for i in `ceph osd pool ls | grep default.rgw`; do ceph osd pool delete $i $i --yes-i-really-really-mean-it; done

radosgw-admin user create --uid="replication-user" --display-name="Multisite v2 replication user" --system
    "keys": [
        {
            "user": "replication-user",
            "access_key": "7AZSXLTKOZRNDFMBUZ4Z",
            "secret_key": "DQ0P1jFUzDQp6wiWg7CrOYheW1ZAy0c1rN7AQRZD"
        }
    ],

radosgw-admin zone modify --rgw-zone=us-east-1 \
--access-key=7AZSXLTKOZRNDFMBUZ4Z \
--secret=DQ0P1jFUzDQp6wiWg7CrOYheW1ZAy0c1rN7AQRZD

radosgw-admin period update --commit

## In cluster2

radosgw-admin realm pull \
--url=http://us-east-1.ceph-s3.com:80 \
--access-key=7AZSXLTKOZRNDFMBUZ4Z \
--secret=DQ0P1jFUzDQp6wiWg7CrOYheW1ZAy0c1rN7AQRZD \
--id rgw.ducpx-s3-cluster2

radosgw-admin realm default --rgw-realm=cookbookv2 --id rgw.ducpx-s3-cluster2

radosgw-admin period pull \
--url=http://us-east-1.ceph-s3.com:80 \
--access-key=7AZSXLTKOZRNDFMBUZ4Z \
--secret=DQ0P1jFUzDQp6wiWg7CrOYheW1ZAy0c1rN7AQRZD \
--id rgw.ducpx-s3-cluster2

radosgw-admin zone create --rgw-zonegroup=us \
--rgw-zone=us-west-1 --access-key=7AZSXLTKOZRNDFMBUZ4Z \
--secret=DQ0P1jFUzDQp6wiWg7CrOYheW1ZAy0c1rN7AQRZD \
--endpoints=http://us-west-1.ceph-s3.com:80 \
--id rgw.ducpx-s3-cluster2
    "system_key": {
            "access_key": "7AZSXLTKOZRNDFMBUZ4Z",
            "secret_key": "DQ0P1jFUzDQp6wiWg7CrOYheW1ZAy0c1rN7AQRZD"
        },

radosgw-admin zone delete --rgw-zone=default

radosgw-admin period update --commit --id rgw.ducpx-s3-cluster2

for i in `ceph osd pool ls --id rgw.ducpx-s3-cluster2 | grep default.rgw`; do ceph osd pool delete $i $i --yes-i-really-really-mean-it --id rgw.ducpx-s3-cluster2; done

systemctl restart ceph-radosgw@rgw.ducpx-s3-cluster2.service

## test

Tao user tren master site

radosgw-admin user create --uid=pratima \
--display-name="Pratima Umrao" \
--id rgw.ducpx-cephaio
{
    "user_id": "pratima",
    "display_name": "Pratima Umrao",
    ...
    "keys": [
        {
            "user": "pratima",
            "access_key": "4E2AYQW87ATALVVIG663",
            "secret_key": "VhVkZ53PAGUjaPMtIhVugjPZVHZl7dBLBNUb5Gk1"
        }
    ],
    ...
}

radosgw-admin metadata list user --id rgw.ducpx-cephaio
radosgw-admin metadata list user --id rgw.ducpx-s3-cluster2
radosgw-admin user info --uid=pratima --id rgw.ducpx-s3-cluster2