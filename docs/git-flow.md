# How to implement Git flow

## Workflows (resuable)
เป็น workflow ที่จะเอาไว้ใช้กับ pipeline หลักอีกทีประกอบไปด้วย 
- `_docker.yaml` - เป็น workflow สำหรับ build images และ push ขึ้น docker hub registry

- `_deploy.yaml` - เป็น workflow สำหรับ apply k8s config ของเราขึ้น production cluster หรือก็คือขั้นตอน deploy นั่นแหละ

- `_testing.yaml` - เป็น workflow สำหรับ run unit test และ integration test

จะอธิบายรายละเอียดใน workflow โดยแบ่ง code ย่อยๆ เพื่อให้เข้าใจเป็นส่วนๆ  และจะมี full version ด้านล่างอีกที

### `_docker.yaml`
- เริ่มจากการ define ว่า workflow นี้เป็น reusable ด้วย keyword `workflow_call`
- โดยจะเตรียมให้ workflow หลักที่เรียกใช้ส่ง input และ secret เข้ามาตามที่เราระบุไว้ถ้าไม่ส่งมาจะพัง

``` yaml title="_docker.yaml"
on:
  workflow_call:
    inputs:
      packages:
        description: Packages to build and publish
        required: true
        type: string
      environment:
        description: Environment to build and publish e.g. prod, beta, dev
        required: true
        type: string
      image-prefix:
        description: Image prefix of the built image
        type: string
      push:
        description: Enable pushing the image to the container registry
        type: boolean
        default: true
    secrets:
      DOCKER_HUB_USERNAME:
        required: true
      DOCKER_HUB_PASSWORD:
        required: true
```


- `inputs`
    - `packages` จะเป็น input ที่บอกว่ามี service ไหนที่จะต้อง build & push ใหม่บ้าง
    ``` json
    [
        {
            "name": "order-service",
            "ref": "refs/tags/order-service@0.0.3-beta.0",
            "imageTag": "0.0.3-beta.0"
        },
        {
            "name": "tenant-service",
            "ref": "refs/tags/tenant-service@0.1.1-beta.0",
            "imageTag": "0.1.1-beta.0"
        }
    ]
    ```
    - `environment` จะเป็น input ที่บอก target environment ของการ build และ push image โดยในที่นี้เรามี 2 environments คือ beta และ prod
    - `image-prefix ` ก็ตรงตัวให้ส่ง image prefix เพื่อมาใช้ต่อ string กับ image tag เพื่อ push ขึ้น registry เราใช้ docker hub ตัว image prefix คือ `goodstockdev`
    - `push` เป็น flag สำหรับใช้ใน action push image ขึ้น registry ซึ่ง default จะเป็น `true`
- `secrets`
    - ใน workflow ที่เป็น reusable workflow จะไม่สามารถเรียก ${{secret.something}} ตรงๆแบบนี้ได้ หรือ อาจจะได้แต่เท่าที่ลองมันจะหาไม่เจอ เราจึงต้อง pass มาให้จาก parent workflow ที่เรียกใช้อีกที
    ``` yaml
        on:
        workflow_call:
        inputs:
        packages:
            description: Packages to build and publish
            required: true
            type: string
        environment:
            description: Environment to build and publish e.g. prod, beta, dev
            required: true
            type: string
        image-prefix:
            description: Image prefix of the built image
            type: string
        push:
            description: Enable pushing the image to the container registry
            type: boolean
            default: true
        secrets:
        DOCKER_HUB_USERNAME:
            required: true
        DOCKER_HUB_PASSWORD:
            required: true
    ```
    - ต่อไปจะเป็นตัว jobs ของ workflow นี้ จะอธิบายเฉพาะ step ที่ดูซับซ้อนและน่า concern
    - ตรงนี้จะทำให้เราสามารถ run job แบบ parallel ได้ ในกรณีที่เรามี 2 services จาก input ด้านบนที่ส่งเข้ามาแล้วเราใช้ `fromJson` เป็นตัว convert ให้เป็น `json` อีกที ซึ่ง step นี้จะทำให้เราสามารถ run job ทั้งหมด สำหรับ 2 services ได้เลย
    ``` yaml
        strategy:
            matrix:
            packages: ${{ fromJson(inputs.packages) }}
    ```
    - เป็นการประกาศเพื่อให้ workflow นี้สามารถส่ง output ออกไปให้ workflow ที่เรียกใช้มันอีกทีเพื่อไปดำเนินการต่อ
    ``` yaml
        outputs:
            DOCKERFILE_EXISTS: ${{ steps.check-dockerfile.outputs.DOCKERFILE_EXISTS }}
    ```
    - ตรงนี้เป็น logic ในการเช็คว่า services ที่ส่งเข้ามามี `Dockerfile` ให้ build หรือเปล่า และจะส่ง boolean เข้า github output ซึ่งเป็นการ set output ให้กับ step ด้านบนด้วย
    ``` yaml
        - name: Check if Dockerfile exists
        id: check-dockerfile
        run: |
            if [ ! -f apps/${{ matrix.packages.name }}/Dockerfile ]; then
            echo "Dockerfile does not exist"
            echo "DOCKERFILE_EXISTS=false" >> $GITHUB_OUTPUT
            else
            echo "Dockerfile exists"
            echo "DOCKERFILE_EXISTS=true" >> $GITHUB_OUTPUT
            fi
    ```
    **Full code version**
    ``` yaml
    name: _docker
    on:
    workflow_call:
        inputs:
        packages:
            description: Packages to build and publish
            required: true
            type: string
        environment:
            description: Environment to build and publish e.g. prod, beta, dev
            required: true
            type: string
        image-prefix:
            description: Image prefix of the built image
            type: string
        push:
            description: Enable pushing the image to the container registry
            type: boolean
            default: true
        secrets:
        DOCKER_HUB_USERNAME:
            required: true
        DOCKER_HUB_PASSWORD:
            required: true
    jobs:
    build-and-publish-docker-image:
        name: Build and Publish Docker image

        runs-on: ubuntu-latest

        strategy:
        matrix:
            packages: ${{ fromJson(inputs.packages) }}

        outputs:
        DOCKERFILE_EXISTS: ${{ steps.check-dockerfile.outputs.DOCKERFILE_EXISTS }}

        steps:
        - name: Checkout with tags
            uses: actions/checkout@v3
            with:
            fetch-depth: 0
            ref: ${{ matrix.packages.ref }}

        - name: Check if Dockerfile exists
            id: check-dockerfile
            run: |
            if [ ! -f apps/${{ matrix.packages.name }}/Dockerfile ]; then
                echo "Dockerfile does not exist"
                echo "DOCKERFILE_EXISTS=false" >> $GITHUB_OUTPUT
            else
                echo "Dockerfile exists"
                echo "DOCKERFILE_EXISTS=true" >> $GITHUB_OUTPUT
            fi

        - name: Set up Docker Buildx
            if: steps.check-dockerfile.outputs.DOCKERFILE_EXISTS == 'true'
            uses: docker/setup-buildx-action@v2

        - name: Login to Docker hub
            if: steps.check-dockerfile.outputs.DOCKERFILE_EXISTS == 'true'
            uses: docker/login-action@v1
            with:
            username: ${{ secrets.DOCKER_HUB_USERNAME }}
            password: ${{ secrets.DOCKER_HUB_PASSWORD }}

        - name: Build and push Docker image
            if: steps.check-dockerfile.outputs.DOCKERFILE_EXISTS == 'true'
            uses: docker/build-push-action@v3
            with:
            context: .
            file: apps/${{ matrix.packages.name }}/Dockerfile
            tags: ${{ inputs.image-prefix }}/${{ matrix.packages.name }}:${{ matrix.packages.imageTag }}
            push: ${{ inputs.push }}
    ```

### `_deploy.yaml`
- ในส่วนนี้ก็เหมือนกับที่อธิบายด้านบนเลยไม่ต่างกัน ขอข้าม step นี้ไป
``` yaml
    on:
    workflow_call:
        inputs:
        packages:
            description: "Packages to update"
            required: true
            type: string
        environment:
            description: "Environment e.g. beta, prod, main used to define path to kustomize overlay"
            required: true
            type: string
        image-prefix:
            description: "Prefix of image e.g. goodstockdev"
            required: true
            type: string
        cluster-name:
            description: "Your cluster name"
            required: true
            type: string
        secrets:
        DIGITALOCEAN_ACCESS_TOKEN:
            required: true
```
- workflow นี้จะใช้ `kustomize` ซึ่งเป็นเครื่องมือในการช่วยจัดการ config k8s ซึ่ง step นี้จะเป็นการเรียกใช้ action `kustomize`
``` yaml
    - name: Setup Kustomize
    uses: multani/action-setup-kustomize@v1
    with:
        version: 5.0.0
```
- step นี้จะทำการ for loop ตัวแปร packages
``` json
    [
        {
            "name": "order-service",
            "ref": "refs/tags/order-service@0.0.3-beta.0",
            "imageTag": "0.0.3-beta.0"
        },
        {
            "name": "tenant-service",
            "ref": "refs/tags/tenant-service@0.1.1-beta.0",
            "imageTag": "0.1.1-beta.0"
        }
    ]
```
```
for row in $(echo "$PACKAGES" | jq -r '.[] | @base64')
``` 
    - โดย คำสั่ง `jq` จะเป็น คำสั่งในการจัดการ `json` โดยเฉพาะ
    - ซึ่ง `-r` จะหมายถึง raw output  เพื่อตัดพวก single quote, double quote, หรือ พวก special character
    - `'.[] | @base64'` อันนี้หมายถึง map ทุกตัวใน array และ เข้ารหัส `base64`

    ```
    _jq() {
        echo ${row} | base64 --decode | jq -r ${1}
    }
    ```
    - และมีการสร้าง function `_jq()` ไว้เพื่อ decode กลับมาเป็น json รูปแบบปกติ
    - ใน loop ก็จะเข้าถึง property object ได้ผ่านการเรียก function เช่น `$(_jq '.name')` เป็นการเรียก property name

    - `bash -c "cd $KUSTOMIZE_PATH && kustomize edit set image $NAME=$PREFIX/$NAME:$IMAGE_TAG"`
        - `-c` อันนี้จะเป็นคำสั่งของ shell ให้เราสามารถ execute command โดยไม่ต้องไปสร้าง shell script
        - ในที่นี้จะมี 2 คำสั่ง คือ `cd` เข้าไปที่ `kustomize` overlay path เพื่อเตรียม edit
        - สั่ง `kustomize edit` ตัว image tag ล่าสุดเพื่อเตรียม apply
        ```
         env:
            PACKAGES: ${{ inputs.packages }}
        ```
        - สังเกตว่าใน for loop จะมีการเรียกใช้ตัวแปร `$PACKAGES` เราเลยต้องส่งตัวแปรนี้เข้าไปผ่าน github env
        ```
        - name: Update kustomize configuration
        run: |
            for row in $(echo "$PACKAGES" | jq -r '.[] | @base64'); do
            _jq() {
                echo ${row} | base64 --decode | jq -r ${1}
            }
            NAME=$(_jq '.name')
            IMAGE_TAG=$(_jq '.imageTag')
            PREFIX=${{ inputs.image-prefix }}
            KUSTOMIZE_PATH=k8s/$NAME/overlays/${{ inputs.environment }}
            [ -d "$KUSTOMIZE_PATH" ] && bash -c "cd $KUSTOMIZE_PATH && kustomize edit set image $NAME=$PREFIX/$NAME:$IMAGE_TAG"
            echo "${NAME}:${IMAGE_TAG} is updated"
            done
        env:
            PACKAGES: ${{ inputs.packages }}
        ```
        - ต่อไปจะเป็น step ของการ commit ไฟล์ที่ถูก kustomize edit จาก step ด้านบนแก้ไข image tag เพื่อให้ตรงกับ concept git ops เราควร track การเปลี่ยนแปลงของไฟล์ให้ได้มากที่สุด step นี้จึงจะ commit ไฟล์ที่ถูกแก้ไขขึ้น git ปกติ
       ```
        - name: Commit image tag changes
        uses: EndBug/add-and-commit@v7
        with:
          add: .
          message: "Update kustomize image latest tag"
       ``` 
       - เดี๋ยวเราจะต้อง login cluster บน digitalocean เพื่อทำการ apply config ใน step ต่อไป ซึ่ง step นี้เราจึงต้อง install `doctl` ซึ่งเป็น tool ที่ช่วยให้เราสามารถ download context config หรือว่า login เข้า cluster นั่นแหละ
       ```
         - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}
       ```
       - step นี้จะเป็นการ download context มาเพื่อเตรียม apply config ในขั้นตอนถัดไป
       ```
        - name: Save DigitalOcean kubeconfig with short-lived credentials
        run: doctl kubernetes cluster kubeconfig save --expiry-seconds 600 ${{inputs.cluster-name}} 
       ```
       - step สุดท้ายคือการ apply config ที่เราเพิ่งแก้ไข image tag ผ่าน kustomize edit โดยเราจะทำการ for loop คล้ายๆ กับ step ด้านบนเพื่อไล่ apply config ให้ครบทุก service ที่ส่งผ่าน input มา
       - โดย logic ใน step นี้จะเหมือนด้านบนเลย มีแตกต่างแค่ command ที่ใช้ในการ apply config ดังนี้ `kustomize build ${{ inputs.environment }} | kubectl apply -f -`
       ```
        - name: Deploy to DigitalOcean Kubernetes cluster
        run: |
          for row in $(echo "$PACKAGES" | jq -r '.[] | @base64'); do
            _jq() {
              echo ${row} | base64 --decode | jq -r ${1}
            }
            NAME=$(_jq '.name')
            KUSTOMIZE_PATH=k8s/$NAME/overlays
            bash -c "cd $KUSTOMIZE_PATH && kustomize build ${{ inputs.environment }} | kubectl apply -f -"
          done
        env:
          PACKAGES: ${{ inputs.packages }}
       ```
       
       **Full Code Version**
       ``` yaml
        name: _deploy
        on:
        workflow_call:
            inputs:
            packages:
                description: "Packages to update"
                required: true
                type: string
            environment:
                description: "Environment e.g. beta, prod, main used to define path to kustomize overlay"
                required: true
                type: string
            image-prefix:
                description: "Prefix of image e.g. goodstockdev"
                required: true
                type: string
            cluster-name:
                description: "Your cluster name"
                required: true
                type: string
            secrets:
            DIGITALOCEAN_ACCESS_TOKEN:
                required: true
        jobs:
        deploy:
            name: Deploy to production

            runs-on: ubuntu-latest

            steps:
            - name: Checkout Repo
                uses: actions/checkout@v3

            - name: Setup Kustomize
                uses: multani/action-setup-kustomize@v1
                with:
                version: 5.0.0

            - name: Update kustomize configuration
                run: |
                for row in $(echo "$PACKAGES" | jq -r '.[] | @base64'); do
                    _jq() {
                    echo ${row} | base64 --decode | jq -r ${1}
                    }
                    NAME=$(_jq '.name')
                    IMAGE_TAG=$(_jq '.imageTag')
                    PREFIX=${{ inputs.image-prefix }}
                    KUSTOMIZE_PATH=k8s/$NAME/overlays/${{ inputs.environment }}
                    [ -d "$KUSTOMIZE_PATH" ] && bash -c "cd $KUSTOMIZE_PATH && kustomize edit set image $NAME=$PREFIX/$NAME:$IMAGE_TAG"
                    echo "${NAME}:${IMAGE_TAG} is updated"
                done
                env:
                PACKAGES: ${{ inputs.packages }}

            - name: Commit image tag changes
                uses: EndBug/add-and-commit@v7
                with:
                add: .
                message: "Update kustomize image latest tag"

            - name: Install doctl
                uses: digitalocean/action-doctl@v2
                with:
                token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

            - name: Save DigitalOcean kubeconfig with short-lived credentials
                run: doctl kubernetes cluster kubeconfig save --expiry-seconds 600 ${{inputs.cluster-name}}

            - name: Deploy to DigitalOcean Kubernetes cluster
                run: |
                for row in $(echo "$PACKAGES" | jq -r '.[] | @base64'); do
                    _jq() {
                    echo ${row} | base64 --decode | jq -r ${1}
                    }
                    NAME=$(_jq '.name')
                    KUSTOMIZE_PATH=k8s/$NAME/overlays
                    bash -c "cd $KUSTOMIZE_PATH && kustomize build ${{ inputs.environment }} | kubectl apply -f -"
                done
                env:
                PACKAGES: ${{ inputs.packages }}
       ```
### `_testing.yaml`
    workflow นี้หลักๆมี 2 jobs คือ 
#### `unit_tests`
เริ่มจาการระบุตัว version `node` และ `os` ที่เราจะใช้ run workflow นี้ การระบุ strategy ตรงนี้จะทำให้ test เราไม่พังเพราะ workflow ไปใช้ version อื่น
``` yaml
    strategy:
      matrix:
        node-version: [18]
        os: [ubuntu-latest]

    runs-on: ${{ matrix.os }}
```
ตรงนี้จะเป็นการ inject `.env` ไฟล์ ทั้งไฟล์ ย้ำว่า ทั้งไฟล์ โดยเราจะเก็บ env ทั้งหมดเข้าไปใน secret ที่เห็นใน code ด้านล่าง ตัวอย่าง `secret.ORDER_SERVICE_TEST_ENV` ซึ่งตรงนี้อาจจะยังไม่ค่อยดีเพราะจะ debug ยากในกรณีที่ต้องการ re-check อาจจะลองพิจารณาพวก external 3rd party ที่ช่วยในการ manage env อีกที
``` yaml
     - name: Inject .env file from secret
        run: |
          echo "${{ secrets.ORDER_SERVICE_TEST_ENV }}" > ./apps/order-service/.env.dev
          echo "${{ secrets.TENANT_SERVICE_TEST_ENV }}" > ./apps/tenant-service/.env.dev
```
เนื่องจาก project เราเป็น `monorepos` และใช้ `turborepo` ในการ manage workspace เราจึงต้องลง cli เพื่อเตรียม run task ใน step ถัดไป
``` yaml
    - name: Install turbo
    run: pnpm install --global turbo
```
ขั้นตอนนี้จะเป็นการ run parallel command ผ่าน `turborepo` task สามารถเช็คได้ที่ไฟล์ `.turbo.json` และดุที่ `package.json` ที่ root directory จะเห็นว่ามีการ define task test:unit ไว้สอดคล้องกัน
``` yaml
    - name: Run unit tests
    run: npm run test:unit
```
#### `integration_tests`
job ของ integration test จะแตกต่างกับ unit test ตรงจะมี spin off database ผ่าน docker compose และ run test ผ่าน docker compose เช่นกัน โดย job นี้จะทำการ up containers ทั้งหมดที่เกี่ยวข้องเตรียมไว้ใน step ถัดไป
``` yaml
 - name: Start Docker-Compose
        run: docker-compose up -d
```
step นี้จะเป็นการรอจนกว่า containers `backend` ด้านบนจะ up เสร็จ
``` yaml
- name: Wait for tenant-service to be ready
        run: |
          while ! nc -z localhost 3002; do
            sleep 1
          done

      - name: Wait for order-service to be ready
        run: |
          while ! nc -z localhost 3001; do
            sleep 1
          done
```
step ถัดไปก็จะเป็นการ run seed database และ integration test ตามลำดับต่อไป

**Full Code Version**
``` yaml
name: "_testing"

on:
  workflow_call:
    secrets:
      ORDER_SERVICE_TEST_ENV:
        required: true
      TENANT_SERVICE_TEST_ENV:
        required: true

jobs:
  unit_tests:
    name: Run unit tests

    strategy:
      matrix:
        node-version: [18]
        os: [ubuntu-latest]

    runs-on: ${{ matrix.os }}

    steps:
      - name: Checkout Repo
        uses: actions/checkout@v2
        with:
          # related to issue, https://github.com/changesets/action/issues/201
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Inject .env file from secret
        run: |
          echo "${{ secrets.ORDER_SERVICE_TEST_ENV }}" > ./apps/order-service/.env.dev
          echo "${{ secrets.TENANT_SERVICE_TEST_ENV }}" > ./apps/tenant-service/.env.dev

      - name: Install turbo
        run: npm install --global turbo

      - name: Run npm install packages
        run: npm install

      - name: Run unit tests
        run: npm run test:unit

  integration_tests:
    name: Run integration tests

    needs: unit_tests

    strategy:
      matrix:
        node-version: [18]
        os: [ubuntu-latest]

    runs-on: ${{ matrix.os }}

    steps:
      - name: Checkout Repo
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}

      - name: Inject .env file from secret
        run: |
          echo "${{ secrets.ORDER_SERVICE_TEST_ENV }}" > ./apps/order-service/.env.dev
          echo "${{ secrets.TENANT_SERVICE_TEST_ENV }}" > ./apps/tenant-service/.env.dev

      - name: Start Docker-Compose
        run: docker-compose up -d

      - name: Wait for tenant-service to be ready
        run: |
          while ! nc -z localhost 3002; do
            sleep 1
          done

      - name: Wait for order-service to be ready
        run: |
          while ! nc -z localhost 3001; do
            sleep 1
          done

      - name: Run seed db
        run: |
          docker-compose exec -T tenant-service npm run seed:db:test

      - name: Run Integration Tests for tenant-service
        run: docker-compose exec -T tenant-service npm run test:integration

      - name: Run Integration Tests for order-service
        run: docker-compose exec -T order-service npm run test:integration

```

## References

- [GitHub - julie-ng/cloud-architecture-review: Cloud Architecture Review App](https://github.com/julie-ng/cloud-architecture-review/tree/main)
- [GitHub - saenyakorn/monorepo-versioning-gitops: Versioning workflows on Monorepo and deploy the services with GitOps concept](https://github.com/saenyakorn/monorepo-versioning-gitops/tree/beta)
- [GitHub - thinc-org/thinc-gitops-example at part-3](https://github.com/thinc-org/thinc-gitops-example/tree/part-3)
- [GitHub - thinc-org/cugetreg: A course registration planning application for CU students](https://github.com/thinc-org/cugetreg/tree/beta)
