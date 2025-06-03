<script setup lang="ts">
import TheWelcome from '../components/TheWelcome.vue'
import { STSClient, AssumeRoleWithWebIdentityCommand } from '@aws-sdk/client-sts'
import { S3Client, ListObjectsV2Command } from '@aws-sdk/client-s3'

async function getTemporaryCredentialsFromServer() {
  const stsClient = new STSClient({
    endpoint: 'https://oss.hubx.pro/',
  })

  // 创建 AssumeRoleWithWebIdentity 命令的参数
  const commandParams = {
    WebIdentityToken:
      'eyJhbGciOiJSUzI1NiIsImtpZCI6Im1pbmlvLW9pZGMta2V5LTIwMjUiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOlsic3RzLmFtYXpvbmF3cy5jb20iXSwiYXpwIjoibWluaW8tY29uc29sZS1jbGllbnQiLCJleHAiOjE3NDkwMjAxODUsImlhdCI6MTc0ODkzMzc4NSwiaXNzIjoiaHR0cHM6Ly9hcGkuaHVieC5wcm8vdjEvYXV0aCIsInBvbGljeSI6ImFkbWluLWZyb20tc3RzLWNsaWVudCIsInN1YiI6IjEiLCJ0ZW5hbnRfaWQiOm51bGx9.f0N0U1ECAQ2Nv_9X3t2rZg11ientl_zPAOP4YELIM3UVk0QzkIF1h1UrjUTUugrObwin08f5BtZ5mh6dghxCJm5v2NIQ-8MO8z4CveGT9l0A-w3tcj43GbafnZx2yGQ0qMU8KWUFVcf_9UrE2XzwEx2oygwkwFCKlgjNkBLBnmmoosqOolENXCFMN3sYwjss3WtOXvN9GSP7xjNMICp5cHLkbjM09iTXRak4DoVif4sGPyeOtghuAQtDd9QBNoV2Wiv4zEGd0D6A8nW5uD2D6DHWIK5VMhmOIGz9tTL5uS5IqY9UBBFlnxJfYgZMY2ay4HFY-3DakCzGTDWiWimqJw',
    RoleArn: '', // 如果需要的话，填写 RoleArn
    RoleSessionName: 'minio-webidentity-session',
    DurationSeconds: 3600,
  }

  try {
    console.log('使用 STS Client 发送 AssumeRoleWithWebIdentityCommand...')
    const command = new AssumeRoleWithWebIdentityCommand(commandParams)
    const data = await stsClient.send(command)

    console.log('成功从 STS Client 获取凭证!')
    // SDK 会自动处理 XML 解析，直接返回结构化的 JS 对象
    return {
      accessKeyId: data.Credentials?.AccessKeyId,
      secretAccessKey: data.Credentials?.SecretAccessKey,
      sessionToken: data.Credentials?.SessionToken,
    }
  } catch (error) {
    console.error('使用 STS Client 获取凭证失败:', error)
    // SDK 提供的错误对象通常包含更丰富的信息
    throw error
  }
}

async function listObjectsWithTempToken(bucketName: string) {
  try {
    console.log('正在从服务器获取临时读取凭证...')
    const tempCredentials = await getTemporaryCredentialsFromServer()

    if (
      !tempCredentials.accessKeyId ||
      !tempCredentials.secretAccessKey ||
      !tempCredentials.sessionToken
    ) {
      console.error('凭证获取失败')
      return
    }

    console.log('凭证获取成功，正在配置 S3 客户端...')
    const s3Client = new S3Client({
      // 如果使用 MinIO，需要配置 endpoint 和 forcePathStyle
      endpoint: 'https://oss.hubx.pro',
      forcePathStyle: true,

      // 通用配置
      region: 'us-east-1',
      credentials: {
        accessKeyId: tempCredentials.accessKeyId,
        secretAccessKey: tempCredentials.secretAccessKey,
        sessionToken: tempCredentials.sessionToken, // 关键：传入会话令牌
      },
    })

    const params = {
      Bucket: bucketName,
      // Prefix: "documents/" // (可选) 如果你只想列出某个目录下的对象，可以加上前缀
    }

    console.log(`正在获取存储桶 "${bucketName}" 中的对象列表...`)
    const command = new ListObjectsV2Command(params)
    const response = await s3Client.send(command)

    console.log('获取成功!')

    // response.Contents 是一个包含对象的数组。如果存储桶为空，它可能是 undefined。
    const objects = response.Contents ?? []

    if (objects.length === 0) {
      console.log('该存储桶中没有对象。')
      return []
    }

    console.log('对象列表:')
    objects.forEach((obj) => {
      console.log(`- Key: ${obj.Key} (大小: ${obj.Size} bytes, 最后修改: ${obj.LastModified})`)
    })

    return objects
  } catch (error) {
    console.error('获取列表失败:', error)
    return []
  }
}
listObjectsWithTempToken('test')
</script>

<template>
  <main>
    <TheWelcome />
  </main>
</template>
